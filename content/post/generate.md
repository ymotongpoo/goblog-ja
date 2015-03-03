+++
date = "2014-12-22T14:49:44+09:00"
draft = false
title = "コードのジェネレート (Generating code)"
+++

# コードのジェネレート
[Generating code](https://blog.golang.org/generate) by Rob Pike

普遍的な計算の性質、チューリング完全、とは、コンピュータプログラムがコンピュータプログラムを書けるということです。
これは、実際に評価されるべきほどには評価されていない、強力な考え方です。たとえば、これはコンパイラの定義の
大きな部分を占めています。また `go test` コマンドの動作にも関わってきます。 `go test` はテスト対象の
パッケージをスキャンして、そのパッケージ用のテストハーネスを含んだGoプログラムを書き出して、
それをコンパイルして実行します。現代のコンパイラは非常に速いので、このコストが高そうな一連の処理も
1秒以内に完了できます。

プログラムがプログラムを書く例は他にもたくさんあります。たとえば [Yacc](http://golang.org/cmd/yacc/) は、
文法の記述を読み込んで、文法をパースするプログラムを書き出します。Protocol Bufferの「コンパイラ」は
インターフェースの記述を読み込んで、構造の定義とメソッドとその他必要なコードを生成します。
あらゆる種類の設定ツールも同様の動作をします。メタデータや環境を確認して、ローカルの状態に合わせた
足場を生成してくれます。

それゆえプログラムを書くプログラムはソフトウェアエンジニアリングにおいて重要な要素ですが、
ソースコードを生成するYaccのようなプログラムは、その出力がコンパイルされるように
ビルドプロセスに組み込まれる必要があります。Makeのような外部のビルドツールが使われる場合ｋ，
これは非常にたやすいことです。しかしGoでは、go toolがソースコードから必要なビルド情報を
すべて取得するので、問題が有ります。go toolだけでYaccを走らせる簡単な機構がないのです。

いままでは、ありませんでした。

## あわせて読みたい

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
* [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
* [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
* [JSON-RPC: a tale of interfaces](https://blog.golang.org/json-rpc-tale-of-interfaces)
