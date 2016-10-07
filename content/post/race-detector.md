+++
date = "2013-06-26T20:38:05+09:00"
draft = true
title = "Goの競合検出機能の紹介（Introducing the Go Race Detector）"

+++

# Goの競合検出機能の紹介

[Introducing the Go Race Detector](https://blog.golang.org/race-detector) by By Dmitry Vyukov and Andrew Gerrand

## はじめに

Race conditions are among the most insidious and elusive programming errors. They typically cause erratic and mysterious failures, often long after the code has been deployed to production. While Go's concurrency mechanisms make it easy to write clean concurrent code, they don't prevent race conditions. Care, diligence, and testing are required. And tools can help.

We're happy to announce that Go 1.1 includes a race detector, a new tool for finding race conditions in Go code. It is currently available for Linux, OS X, and Windows systems with 64-bit x86 processors.

The race detector is based on the C/C++ ThreadSanitizer runtime library, which has been used to detect many errors in Google's internal code base and in Chromium. The technology was integrated with Go in September 2012; since then it has detected 42 races in the standard library. It is now part of our continuous build process, where it continues to catch race conditions as they arise.

## How it works

The race detector is integrated with the go tool chain. When the -race command-line flag is set, the compiler instruments all memory accesses with code that records when and how the memory was accessed, while the runtime library watches for unsynchronized accesses to shared variables. When such "racy" behavior is detected, a warning is printed. (See this article for the details of the algorithm.)

Because of its design, the race detector can detect race conditions only when they are actually triggered by running code, which means it's important to run race-enabled binaries under realistic workloads. However, race-enabled binaries can use ten times the CPU and memory, so it is impractical to enable the race detector all the time. One way out of this dilemma is to run some tests with the race detector enabled. Load tests and integration tests are good candidates, since they tend to exercise concurrent parts of the code. Another approach using production workloads is to deploy a single race-enabled instance within a pool of running servers.

## Using the race detector

The race detector is fully integrated with the Go tool chain. To build your code with the race detector enabled, just add the -race flag to the command line:

```
$ go test -race mypkg    // test the package
$ go run -race mysrc.go  // compile and run the program
$ go build -race mycmd   // build the command
$ go install -race mypkg // install the package
```

To try out the race detector for yourself, fetch and run this example program:

```
$ go get -race golang.org/x/blog/support/racy
$ racy
```

## Examples

Here are two examples of real issues caught by the race detector.

### Example 1: Timer.Reset

The first example is a simplified version of an actual bug found by the race detector. It uses a timer to print a message after a random duration between 0 and 1 second. It does so repeatedly for five seconds. It uses time.AfterFunc to create a Timer for the first message and then uses the Reset method to schedule the next message, re-using the Timer each time.

```
func main() {
    start := time.Now()
    var t *time.Timer
    t = time.AfterFunc(randomDuration(), func() {
        fmt.Println(time.Now().Sub(start))
        t.Reset(randomDuration())
    })
    time.Sleep(5 * time.Second)
}

func randomDuration() time.Duration {
    return time.Duration(rand.Int63n(1e9))
}
```

実行結果（毎回変化します）

```
947.77941ms
1.029932961s
1.696078782s
1.931088833s
2.21820277s
2.76737009s
3.400339848s
3.732115996s
3.915233212s
4.395512661s

Program exited.
```

This looks like reasonable code, but under certain circumstances it fails in a surprising way:

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xb code=0x1 addr=0x8 pc=0x41e38a]

goroutine 4 [running]:
time.stopTimer(0x8, 0x12fe6b35d9472d96)
    src/pkg/runtime/ztime_linux_amd64.c:35 +0x25
time.(*Timer).Reset(0x0, 0x4e5904f, 0x1)
    src/pkg/time/sleep.go:81 +0x42
main.func·001()
    race.go:14 +0xe3
created by time.goFunc
    src/pkg/time/sleep.go:122 +0x48
```

What's going on here? Running the program with the race detector enabled is more illuminating:

```
==================
WARNING: DATA RACE
Read by goroutine 5:
  main.func·001()
     race.go:14 +0x169

Previous write by goroutine 1:
  main.main()
      race.go:15 +0x174

Goroutine 5 (running) created at:
  time.goFunc()
      src/pkg/time/sleep.go:122 +0x56
  timerproc()
     src/pkg/runtime/ztime_linux_amd64.c:181 +0x189
==================
```

The race detector shows the problem: an unsynchronized read and write of the variable t from different goroutines. If the initial timer duration is very small, the timer function may fire before the main goroutine has assigned a value to t and so the call to t.Reset is made with a nil t.

To fix the race condition we change the code to read and write the variable t only from the main goroutine:

```
func main() {
    start := time.Now()
    reset := make(chan bool)
    var t *time.Timer
    t = time.AfterFunc(randomDuration(), func() {
        fmt.Println(time.Now().Sub(start))
        reset <- true
    })
    for time.Since(start) < 5*time.Second {
        <-reset
        t.Reset(randomDuration())
    }
}
```

Here the main goroutine is wholly responsible for setting and resetting the Timer t and a new reset channel communicates the need to reset the timer in a thread-safe way.

A simpler but less efficient approach is to avoid reusing timers.

### Example 2: ioutil.Discard

The second example is more subtle.

The ioutil package's Discard object implements io.Writer, but discards all the data written to it. Think of it like /dev/null: a place to send data that you need to read but don't want to store. It is commonly used with io.Copy to drain a reader, like this:

```
io.Copy(ioutil.Discard, reader)
```

Back in July 2011 the Go team noticed that using Discard in this way was inefficient: the Copy function allocates an internal 32 kB buffer each time it is called, but when used with Discard the buffer is unnecessary since we're just throwing the read data away. We thought that this idiomatic use of Copy and Discard should not be so costly.

The fix was simple. If the given Writer implements a ReadFrom method, a Copy call like this:

```
io.Copy(writer, reader)
```

is delegated to this potentially more efficient call:

```
writer.ReadFrom(reader)
```

We added a ReadFrom method to Discard's underlying type, which has an internal buffer that is shared between all its users. We knew this was theoretically a race condition, but since all writes to the buffer should be thrown away we didn't think it was important.

When the race detector was implemented it immediately flagged this code as racy. Again, we considered that the code might be problematic, but decided that the race condition wasn't "real". To avoid the "false positive" in our build we implemented a non-racy version that is enabled only when the race detector is running.

But a few months later Brad encountered a frustrating and strange bug. After a few days of debugging, he narrowed it down to a real race condition caused by ioutil.Discard.

Here is the known-racy code in io/ioutil, where Discard is a devNull that shares a single buffer between all of its users.

```
var blackHole [4096]byte // shared buffer

func (devNull) ReadFrom(r io.Reader) (n int64, err error) {
    readSize := 0
    for {
        readSize, err = r.Read(blackHole[:])
        n += int64(readSize)
        if err != nil {
            if err == io.EOF {
                return n, nil
            }
            return
        }
    }
}
```

Brad's program includes a trackDigestReader type, which wraps an io.Reader and records the hash digest of what it reads.

```
type trackDigestReader struct {
    r io.Reader
    h hash.Hash
}

func (t trackDigestReader) Read(p []byte) (n int, err error) {
    n, err = t.r.Read(p)
    t.h.Write(p[:n])
    return
}
```

For example, it could be used to compute the SHA-1 hash of a file while reading it:

```
tdr := trackDigestReader{r: file, h: sha1.New()}
io.Copy(writer, tdr)
fmt.Printf("File hash: %x", tdr.h.Sum(nil))
```

In some cases there would be nowhere to write the data—but still a need to hash the file—and so Discard would be used:

```
io.Copy(ioutil.Discard, tdr)
```

But in this case the blackHole buffer isn't just a black hole; it is a legitimate place to store the data between reading it from the source io.Reader and writing it to the hash.Hash. With multiple goroutines hashing files simultaneously, each sharing the same blackHole buffer, the race condition manifested itself by corrupting the data between reading and hashing. No errors or panics occurred, but the hashes were wrong. Nasty!

```
func (t trackDigestReader) Read(p []byte) (n int, err error) {
    // the buffer p is blackHole
    n, err = t.r.Read(p)
    // p may be corrupted by another goroutine here,
    // between the Read above and the Write below
    t.h.Write(p[:n])
    return
}
```

The bug was finally fixed by giving a unique buffer to each use of ioutil.Discard, eliminating the race condition on the shared buffer.

## Conclusions

The race detector is a powerful tool for checking the correctness of concurrent programs. It will not issue false positives, so take its warnings seriously. But it is only as good as your tests; you must make sure they thoroughly exercise the concurrent properties of your code so that the race detector can do its job.

What are you waiting for? Run "go test -race" on your code today!

By Dmitry Vyukov and Andrew Gerrand

## あわせて読みたい

* [Generating code](https://blog.golang.org/generate)
* [Go Concurrency Patterns: Context](https://blog.golang.org/context)
* [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)
* [Advanced Go Concurrency Patterns](https://blog.golang.org/advanced-go-concurrency-patterns)
* [Go maps in action](https://blog.golang.org/go-maps-in-action)
* [go fmt your code](https://blog.golang.org/go-fmt-your-code)
* [Concurrency is not parallelism](https://blog.golang.org/concurrency-is-not-parallelism)
* [Organizing Go code](https://blog.golang.org/organizing-go-code)
* [Go videos from Google I/O 2012](https://blog.golang.org/go-videos-from-google-io-2012)
* [Debugging Go programs with the GNU Debugger](https://blog.golang.org/debugging-go-programs-with-gnu-debugger)
* [The Go image/draw package](https://blog.golang.org/go-imagedraw-package)
* [The Go image package](https://blog.golang.org/go-image-package)
* [The Laws of Reflection](https://blog.golang.org/laws-of-reflection)
* [Error handling and Go](https://blog.golang.org/error-handling-and-go)
* ["First Class Functions in Go"](https://blog.golang.org/first-class-functions-in-go-and-new-go)
* [Profiling Go Programs](https://blog.golang.org/profiling-go-programs)
* [A GIF decoder: an exercise in Go interfaces](https://blog.golang.org/gif-decoder-exercise-in-go-interfaces)
* [Introducing Gofix](https://blog.golang.org/introducing-gofix)
* [Godoc: documenting Go code](https://blog.golang.org/godoc-documenting-go-code)
* [Gobs of data](https://blog.golang.org/gobs-of-data)
* [C? Go? Cgo!](https://blog.golang.org/c-go-cgo)
* [JSON and Go](https://blog.golang.org/json-and-go)
* [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
* [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
* [JSON-RPC: a tale of interfaces](https://blog.golang.org/json-rpc-tale-of-interfaces)