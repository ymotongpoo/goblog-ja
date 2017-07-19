+++
date = "2011-03-17T16:49:45+09:00"
title = "C? Go? Cgo!"
draft = false
tags = ["cgo"]
+++

# C? Go? Cgo!
[C? Go? Cgo!](https://blog.golang.org/c-go-cgo) By Andrew Gerrand

## はじめに

cgoを使うことでGoパッケージを通してCコードを呼び出すことができます。ある特徴を持って書かれたGoのソースファイルが与えられると、cgoは単一のGoパッケージにまとめることが可能なGoとCのファイルを出力します。

例として、ここにCの関数 `random` と `srandom` をラップした2つの関数 - `Random` と `Seed` - を提供するGoパッケージがあります。

```
package rand

/*
#include <stdlib.h>
*/
import "C"

func Random() int {
    return int(C.random())
}

func Seed(i int) {
    C.srandom(C.uint(i))
}
```

ここで何が起きているか見ていきましょう、importの文から見ていきます。

`rand` パッケージは `"C"` をインポートしていますが、Goのライブラリにはそのようなパッケージがないことに気づくでしょう。

`rand` パッケージは `C` パッケージに対する４つの参照を含んでいます: `C.random` と `C.srandom` の呼び出しと、 変換 `C.uint(i)` 、そして `import` 文です。

`Random` 関数はC標準ライブラリの `random` 関数を呼び出し、その結果を返しています。C内で、`random` はcgoにおける `C.long` 型である `lang` 型の値を返しています。それはこのパッケージ外部のGoコードで使われる前に、通常のGoの型変換によってGoの型に変換しなければなりません:

```
func Random() int {
    return int(C.random())
}
```

型変換を明示的に示すための一時変数を用いた同様の関数は以下のとおりです:

```
func Random() int {
    var r C.long = C.random()
    return int(r)
}
```

`Seed` 関数はある意味で逆のことをします。その関数は正規のGoの型 `int` を引数にとり、それをCの型 `unsigned int` に変換し、Cの関数 `srandom` に渡します。

```
func Seed(i int) {
    C.srandom(C.uint(i))
}
```

cgoは `unsigned int` を `C.uint` として認識していることに注意してください。数値型名の完全なリストは [cgo documentation](http://golang.org/cmd/cgo) をご覧ください。

私たちがこの例においてまだ考察していない詳細部分は、 `import` 文の上にあるコメントです。

```
/*
#include <stdlib.h>
*/
import "C"
```

cgoはこのコメントを認識してます。スペースに続く `#cgo` で始まる行は全て取り除かれます。それらはcgoのための指示子となります。パッケージのCの部分をコンパイルする際に残りの行はヘッダーとして用いられます。この場合それらの行はちょうど単一の `#include` 文となりますが、それらはほぼ全てのCコードとなりえます。パッケージのCの部分をビルドする際に、コンパイラやリンカのためのフラグを提供するために `#cgo` が使われます。

制限があります: `//export` 指示子を使用する場合、コメント中のCコードは宣言（`extern int f();`）のみ含むことができ、定義（`int f() { return 1; }`）を含むことができません。Cコードにアクセス可能なGoコードを作るために `//export` 指示子を使うことができます。

`#cgo` と `//export` 指示子に関するドキュメントは[cgo documentation](http://golang.org/cmd/cgo/)にあります。

## Strings と things

Goと違い、Cには明示的なstring型がありません。Cにおける文字列は、NULL終端された文字配列によって表されます。

GoとCの間の文字列変換は、`C.String` と `C.GoString` 、そして `C.GoStringN` 関数を用いて行うことができます。それらの変換は文字列データのコピーを生成します。

以下の例はCの `stdio` ライブラリの `fputs` 関数を用いて標準出力に文字列を書き込む `Print` 関数の実装です:

```
package print

// #include <stdio.h>
// #include <stdlib.h>
import "C"
import "unsafe"

func Print(s strics := C.CString(s)
    cs := C.CString(s)
    C.fputs(cs, (*C.FILE)(C.stdout))
    C.free(unsafe.Pointer(cs}
}
```

Cコードによって生成されたメモリアロケーションは、Goのメモリマネージャーから認識されません。`C.String` （または任意のCのメモリアロケーション）を用いてCの文字列を生成するときは、`C.free` を呼び出してメモリを解放することを忘れてはいけません。

`C.CString` の呼び出しは文字配列の最初を示すポインタを返すので、関数から抜け出す前にそのポインタを [unsafe.Pointer](http://golang.org/pkg/unsafe/#Pointer) に変換し、`C.free` を用いてメモリアロケーションを解放します。cgoプログラムにおける共通のイディオムは、この書き換えた `Print` のように、アロケートしたあとすぐに [defer](http://golang.org/doc/articles/defer_panic_recover.html) によって解放することです（特にあとに続くコードが一つの関数呼び出しより複雑な場合）:

```
func Print(s string) {
    cs := C.CString(s)
    defer C.free(unsafe.Pointer(cs))
    C.fputs(cs, (*C.FILE)(C.stdout))
}
```

## cgoのパッケージをビルドする

cgoのパッケージをビルドするために、いつもどおり [go build](http://golang.org/cmd/go/#Compile_packages_and_dependencies) または [go install](http://golang.org/cmd/go/#Compile_and_install_packages_and_dependencies) を使いましょう。go toolは特別な `"C"` インポートを認識し、それらのファイルのために自動的にcgoを使います。

## さらなるcgoのリソース

[cgo command](http://golang.org/cmd/cgo/) のドキュメントにCの擬似パッケージやそのビルドプロセスについてより詳しく載っています。Goディレクトリツリー内の [cgo examples](http://golang.org/misc/cgo/) により高度な発想が示されています。

最後に、これが内部でどのように動いているか興味がございましたら、ランタイムパッケージの [cgocall.go](https://golang.org/src/runtime/cgocall.go) の先頭のコメントをご覧ください。

*By Andrew Gerrand*

## あわせて読みたい
* [HTTP/2 Server Push](https://blog.golang.org/h2push)
* [Introducing HTTP Tracing](https://blog.golang.org/http-tracing)
* [Generating code](https://blog.golang.org/generate)
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
* [JSON and Go](https://blog.golang.org/json-and-go)
* [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
* [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
* [JSON-RPC: a tale of interfaces](https://blog.golang.org/json-rpc-tale-of-interfaces)
