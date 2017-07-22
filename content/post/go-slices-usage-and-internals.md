+++
date = "2011-01-05T20:57:09+09:00"
title = "Goのスライス: 使い方と内部詳細（Go Slices: usage and internals）"
draft = false
tags = ["slice", "technical"]
+++

# Goのスライス: 使い方と内部詳細
[Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals) By Andrew Gerrand

## はじめに

Goのスライス型は型付きのデータ列を伴って動作する便利で効率的な手段を提供します。スライスは他の言語における配列に似ていますが、いくつか珍しい特性を持っています。この記事ではスライスが何であるか、そしてどうやって使うかについて見ていきます。

## 配列

スライス型はGoの配列型を抽象化したものなので、スライスを理解する前にまずは配列を理解しなければなりません。

配列型は長さと要素型を明示的に定義します。例えば、型 `[4]int` は4つの整数から成る配列を表しています。配列のサイズは固定されており、その長さはその型の一部となっています（`[4]int` と `[5]int` は全く別の相いれない型です）。配列はいつも通りのやり方でインデックス付けることができるので、`s[n]` はn番目の要素（0スタート）にアクセスします。

```
var a [4]int
a[0] = 1
i := a[0]
// i == 1
```

配列は明示的に初期化する必要がありません。配列のゼロ値は、ゼロ値初期化された要素を持つすぐに使える配列です:

```
// a[2] == 0, the zero value of the int type
```

`[4]int` のメモリ上での表現は、連続して割り付けられたちょうど4つの整数値になっています:

![go-slices-usage-and-internals_slice-array](./go-slices-usage-and-internals_slice-array.png)

Goの配列は値です。配列変数は配列全体を示しています。（Cのように）最初の配列要素を指すポインタではありません。これは配列の値を割り当てたり分配するとき、その内容のコピーを作ることを意味しています（コピーを避けるために配列に*ポインタ*を渡すことができますが、それは配列ではなく配列へのポインタになります）。配列は、名前付けられたフィールドというよりインデックス付けされた構造体の一種、すなわち固定長の複合値として考えられます。

配列のリテラルは以下のように書けます:

```
b := [2]string{"Penn", "Teller"}
```

また、配列の要素数の記述をコンパイラに任せることもできます:

```
b := [...]string{"Penn", "Teller"}
```

どちらの場合でも、`b` の型は `[2]string` です。

## スライス

配列は自身の地位を獲得していますが、少し柔軟性に欠けるのでGoコードでは配列をあまり見ることはないでしょう。一方で、スライスはいたるところで見かけるでしょう。スライスは巨大な力や便利さを提供するため配列を元にしています。

スライスの型指定は `[]T` で行い、`T` はスライスの要素の型です。配列型と違い、スライス型には明示的な長さがありません。

スライスのリテラルは、配列の要素数を除いてちょうど配列のリテラルのように宣言されます:

```
letters := []string{"a", "b", "c", "d"}
```

スライスは `make` と呼ばれるシグネチャを持ったビルトイン関数を用いて作ることができます、

```
func make([]T, len, cap) []T
```

ここでTは作成するスライスの要素型を表しています。`make` 関数は型、長さ、そしてオプションで容量を引数に取ります。`make` が呼ばれると、配列をアロケートし、配列を参照するスライスを返します。

```
var s []byte
s = make([]byte, 5, 5)
// s == []byte{0, 0, 0, 0, 0}
```

引数の容量が省略されると、指定した長さがデフォルトで設定されます。以下は同じコードのより簡潔なバージョンです:

```
s := make([]byte, 5)
```

スライスの長さと容量はビルトイン関数の `len` と `map` を用いて調べることができます。

```
len(s) == 5
cap(s) == 5
```

続く2つの節では、長さと容量の関係について議論していきます。

スライスのゼロ値は `nil` です。スライスがnilの場合、`len` と `cap` 関数は両方とも0を返します。

スライスは既にあるスライスや配列を"スライシング"することによって作ることもできます。スライシングはコロンによって区切られた2のインデックスを用いた半開区間を明示することによって行われます。例えば、式 `b[1:4]` は `b` の1から3番目の要素を含むスライスを作ります（結果のスライスのインデックスは0から2までです）。

```
b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
// b[1:4] == []byte{'o', 'l', 'a'}, sharing the same storage as b
```

スライスの最初と最後のインデックスはオプションです。それらはデフォルトではそれぞれ0とスライスの長さ設定されます:

```
// b[:2] == []byte{'g', 'o'}
// b[2:] == []byte{'l', 'a', 'n', 'g'}
// b[:] == b
```

これも配列を与えてスライスを作るための構文です:

```
x := [3]string{"Лайка", "Белка", "Стрелка"}
s := x[:] // a slice referencing the storage of x
```

## スライスの内部詳細

スライスは配列セグメントの記述子です。記述子は、配列へのポインタptr、セグメント長len、スライスの容量cap（セグメントの最大長）を含んでいます。

![go-slices-usage-and-internals_slice-struct](./go-slices-usage-and-internals_slice-struct.png)

少し前に `make([]byte, 5)` により作った変数 `s` はこのような構成になっています:

![go-slices-usage-and-internals_slice-1](./go-slices-usage-and-internals_slice-1.png)

長さはスライスによって参照される要素の数です。容量は（スライスのポインタが参照する要素から始まる）元の配列の要素の数です。長さと容量の違いは、次のいくつかの例を通して明らかになるでしょう。

`s` をスライスして、スライスデータの構造や元の配列との関係の変化を観察します:

```
s = s[2:4]
```

![go-slices-usage-and-internals_slice-2](./go-slices-usage-and-internals_slice-2.png)

スライシングではスライスデータをコピーしません。元の配列を指し示す新しいスライス値を作ります。これは配列のインデックス操作と同じくらい効率的なスライス操作を作ります。したがって、スライシングによるスライスの*要素*（スライス自身ではありません）を書き換えると元のスライスの要素も書き換わります:

```
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:] 
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

以前私たちは `s` をその容量より小さな長さでスライスしました。私たちは再びスライシングすることによってsをその容量まで大きくすることができます:

```
s = s[:cap(s)]
```

![go-slices-usage-and-internals_slice-3](./go-slices-usage-and-internals_slice-3.png)

A slice cannot be grown beyond its capacity. Attempting to do so will cause a runtime panic, just as when indexing outside the bounds of a slice or array. Similarly, slices cannot be re-sliced below zero to access earlier elements in the array.

スライスはその容量を超えて大きくすることはできません。もしそうした場合、スライスや配列の範囲外を参照したときのようにランタイムパニックが起こるでしょう。同じように、配列の最初の要素より前にアクセスするためにスライスを0以下に再スライスすることはできません。

## スライスを大きくする（コピー・アペンド関数）

To increase the capacity of a slice one must create a new, larger slice and copy the contents of the original slice into it. This technique is how dynamic array implementations from other languages work behind the scenes. The next example doubles the capacity of `s` by making a new slice, `t`, copying the contents of `s` into `t`, and then assigning the slice value `t` to `s`:

```
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

The looping piece of this common operation is made easier by the built-in copy function. As the name suggests, copy copies data from a source slice to a destination slice. It returns the number of elements copied.

```
func copy(dst, src []T) int
```

The `copy` function supports copying between slices of different lengths (it will copy only up to the smaller number of elements). In addition, `copy` can handle source and destination slices that share the same underlying array, handling overlapping slices correctly.

Using `copy`, we can simplify the code snippet above:

```
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

A common operation is to append data to the end of a slice. This function appends byte elements to a slice of bytes, growing the slice if necessary, and returns the updated slice value:

```
func AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { // if necessary, reallocate
        // allocate double what's needed, for future growth.
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n]
    copy(slice[m:n], data)
    return slice
}
```

One could use `AppendByte` like this:

```
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

Functions like `AppendByte` are useful because they offer complete control over the way the slice is grown. Depending on the characteristics of the program, it may be desirable to allocate in smaller or larger chunks, or to put a ceiling on the size of a reallocation.

But most programs don't need complete control, so Go provides a built-in `append` function that's good for most purposes; it has the signature

```
func append(s []T, x ...T) []T 
```

The `append` function appends the elements `x` to the end of the slice `s`, and grows the slice if a greater capacity is needed.

```
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

To append one slice to another, use `...` to expand the second argument to a list of arguments.

```
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

Since the zero value of a slice (`nil`) acts like a zero-length slice, you can declare a slice variable and then append to it in a loop:

```
// Filter returns a new slice holding only
// the elements of s that satisfy f()
func Filter(s []int, fn func(int) bool) []int {
    var p []int // == nil
    for _, v := range s {
        if fn(v) {
            p = append(p, v)
        }
    }
    return p
}
```

## "gotcha" できる

As mentioned earlier, re-slicing a slice doesn't make a copy of the underlying array. The full array will be kept in memory until it is no longer referenced. Occasionally this can cause the program to hold all the data in memory when only a small piece of it is needed.

For example, this `FindDigits` function loads a file into memory and searches it for the first group of consecutive numeric digits, returning them as a new slice.

```
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

This code behaves as advertised, but the returned `[]byte` points into an array containing the entire file. Since the slice references the original array, as long as the slice is kept around the garbage collector can't release the array; the few useful bytes of the file keep the entire contents in memory.

To fix this problem one can copy the interesting data to a new slice before returning it:

```
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

A more concise version of this function could be constructed by using `append`. This is left as an exercise for the reader.

## 関連記事

[Effective Go](http://golang.org/doc/effective_go.html) contains an in-depth treatment of [slices](http://golang.org/doc/effective_go.html#slices) and [arrays](http://golang.org/doc/effective_go.html#arrays), and the Go [language specification](http://golang.org/doc/go_spec.html) defines [slices](http://golang.org/doc/go_spec.html#Slice_types) and their [associated](http://golang.org/doc/go_spec.html#Length_and_capacity) [helper](http://golang.org/doc/go_spec.html#Making_slices_maps_and_channels) [functions](http://golang.org/doc/go_spec.html#Appending_and_copying_slices).

[Effective Go](http://golang.org/doc/effective_go.html) には [スライス](http://golang.org/doc/effective_go.html#slices) や [配列](http://golang.org/doc/effective_go.html#arrays) の詳細な扱い方が載っており、Go [Language Specification](http://golang.org/doc/go_spec.html) では [スライス](http://golang.org/doc/go_spec.html#Slice_types) とそれに [関連した](http://golang.org/doc/go_spec.html#Length_and_capacity) [ヘルパー](http://golang.org/doc/go_spec.html#Making_slices_maps_and_channels) [関数](http://golang.org/doc/go_spec.html#Appending_and_copying_slices) が定義されています。

## あわせて読みたい

* [HTTP/2 Server Push](https://blog.golang.org/h2push)
* [Introducing HTTP Tracing](https://blog.golang.org/http-tracing)
* [Generating code](https://blog.golang.org/generate)
* [Arrays, slices (and strings): The mechanics of 'append'](https://blog.golang.org/slices)
* [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
* [Go maps in action](https://blog.golang.org/go-maps-in-action)
* [go fmt your code](https://blog.golang.org/go-fmt-your-code)
* [Organizing Go code](https://blog.golang.org/organizing-go-code)
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
* [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
* [JSON-RPC: a tale of interfaces](https://blog.golang.org/json-rpc-tale-of-interfaces)