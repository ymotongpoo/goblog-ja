+++
date = "2010-09-23T15:02:14+09:00"
draft = false
title = "Goの並行パターン：タイムアウトと進行（Go Concurrency Patterns: Timing out, moving on）"
tags = ["concurrency", "technical"]
+++

# Goの並行パターン：タイムアウトと進行

[Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and) by Andrew Gerrand

並行プログラミングにはイディオムがあります。良い例はタイムアウトです。
Goのチャンネルではタイムアウトを直接はサポートしていませんが、その実装は容易です。
たとえば、`ch` チャンネルから値を受信したいけれど、1秒以上は待ちたくないという状況を考えてみましょう。
まずシグナル用のチャンネルを作り、そのチャンネルに送信する前に1秒待つゴルーチンを起動します。

```
timeout := make(chan bool, 1)
go func() {
    time.Sleep(1 * time.Second)
    timeout <- true
}()
```

その後、`select`構文を使って `ch` か `timeout` を待つようにします。
もし1秒待っても `ch` から何も来なければ、 `timeout` のケースが選択され、`ch`からの読み込みは破棄されます。

```
select {
case <-ch:
    // chから読み込む
case <-timeout:
    // chからの読み込みはタイムアウト
}
```

`timeout` チャンネルは1つの値をバッファし、タイムアウトのゴルーチンがそのチャンネルに値を送り、終了できるようになっています。
このゴルーチンは、`ch`から値が受け取られたかを知りません。（あるいは気にしていません）
つまりこのゴルーチンは`ch`からの読み込みがタイムアウトより前に起こったとしても、永遠には存在しえません。
`timeout` チャンネルは最終的にガベージコレクタによって回収されます。

（この例ではゴルーチンとチャンネルの機構をデモするために `time.Sleep` を使いました。
実際のプログラムでは `time.After` という、チャンネルを返し、決まった時間のあとにそのチャンネルに値を送る関数を使うべきでしょう。）

このパターンの他の例を見てみましょう。この例では複数のレプリケーションされたデータベースから同時に読み込むプログラムを扱っています。
このプログラムでは、値は1つだけ必要で最初に来た値だけを取得すべきです。

`Query` 関数はデータベース接続のスライスと問い合わせの文字列を引数に取ります。
この関数は各データベースに並列に問い合わせ、最初に受信した結果を返します。

```
func Query(conns []Conn, query string) Result {
    ch := make(chan Result, 1)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}
```

この例では、クロージャーがノンブロッキングに送信します。これは、 `default` ケース付きの `select` 構文内の送信操作を使うことで実現しています。
もし送信が出来なければ、直ちに `default` ケースが選択されます。
送信をノンブロッキングにすることで、ループ内で立ち上げられたゴルーチンが1つも無駄に生存しないことが保証されます。
しかしながら、親の関数が値を受信する前に結果が来れば、チャンネルのバッファの準備ができていないため送信は失敗する可能性があります。

この問題は[競合条件](https://en.wikipedia.org/wiki/Race_condition)として知られるものの教科書的な例ですが、修正は些細なものです。
`ch` チャンネルを（バッファの長さをmakeの第2引数に加えることで）バッファして、最初の送信処理が値を送れるように保証すれば良いだけです。
これによって送信処理は常に成功し、実行順に関係なく最初に到着した値が受信されるようになります。

この2つの例はGoがゴルーチン間の複雑なやりとりを表現する際の簡潔さを表しています。

By Andrew Gerrand

## あわせて読みたい
* [Generating code](https://blog.golang.org/generate)
  * [コードのジェネレート](../generate/)
* [Go Concurrency Patterns: Context](https://blog.golang.org/context)
  * [Goの並行パターン：コンテキスト](../context/)
* [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)
  * [Goの並行パターン：パイプラインとキャンセル](../pipelines/)
* [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
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
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
* [JSON-RPC: a tale of interfaces](https://blog.golang.org/json-rpc-tale-of-interfaces)
