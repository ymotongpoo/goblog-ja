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

![go-slices-usage-and-internals_slice-array](./go-slices-usage-and-internals/go-slices-usage-and-internals_slice-array.png)

Goの配列は値です。配列変数は配列全体を示しています。（Cのように）最初の配列要素を指すポインタではありません。これは配列の値を割り当てたり分配するとき、その中身のコピーを作ることを意味しています（コピーを避けるために配列に*ポインタ*を渡すことができますが、それは配列ではなく配列へのポインタになります）。配列は、名前付けられたフィールドというよりインデックス付けされた構造体の一種、すなわち固定長の複合値として考えられます。

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

スライスの最初と最後のインデックスはオプションです。それらはデフォルトではそれぞれ0とスライス長が設定されます:

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

![go-slices-usage-and-internals_slice-struct](./go-slices-usage-and-internals/go-slices-usage-and-internals_slice-struct.png)

少し前に `make([]byte, 5)` から作った変数 `s` はこのような構成になっています:

![go-slices-usage-and-internals_slice-1](./go-slices-usage-and-internals/go-slices-usage-and-internals_slice-1.png)

長さはスライスによって参照される要素の数です。容量は（スライスのポインタが参照する要素から始まる）元の配列の要素の数です。長さと容量の違いは、次のいくつかの例を通して明らかになるでしょう。

`s` をスライスして、スライスデータの構造や元の配列との関係の変化を観察します:

```
s = s[2:4]
```

![go-slices-usage-and-internals_slice-2](./go-slices-usage-and-internals/go-slices-usage-and-internals_slice-2.png)

スライシングではスライスデータをコピーしません。元の配列を指し示す新しいスライス値を作ります。これは配列のインデックス操作と同じくらい効率的なスライス操作を作ります。したがって、スライシングによるスライスの*要素*（スライス自身ではありません）を書き換えると元のスライスの要素も書き換わります:

```
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:] 
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

さきほど私たちは `s` をその容量より小さな長さでスライスしました。私たちは再びスライシングすることによってsをその容量まで拡張できます:

```
s = s[:cap(s)]
```

![go-slices-usage-and-internals_slice-3](./go-slices-usage-and-internals/go-slices-usage-and-internals_slice-3.png)

スライスはその容量を超えて拡張することはできません。もしそうした場合、スライスや配列の範囲外を参照したときのようにランタイムパニックが起こるでしょう。同じように、配列の最初の要素より前にアクセスするためにスライスを0以下に再スライスすることはできません。

## スライスを拡張する（コピー・アペンド関数）

スライスの容量を増やすために、新しくてより大きいスライスを作り、元のスライスの中身をその中にコピーするでしょう。このテクニックは他言語の動的配列の実装が表立たないところで行う手法です。
次の例では、`s` の容量を2倍にするために、新しいスライス `t` を作り、`s` の中身を `t` にコピーし、スライスの値 `t` を `s` に割り当てています:

```
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

このループの部分の一般的な操作は、ビルトインのcopy関数によって簡略化できます。名前が示す通り、copyは元のスライスから目的のスライスにデータをコピーします。そして、copyはコピーした要素の数を返します。

```
func copy(dst, src []T) int
```

`copy` 関数は異なる長さのスライス間でのコピーに対応しています（小さい方の要素数だけコピーします）。さらに、`copy` は元となる同じ配列を共有する元のスライスと目的のスライスを扱い、オーバーラップするスライスを正しく扱います。

`copy` を使い、上記のコードの断片を簡略化できます:

```
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

一般的な操作のひとつはデータをスライスの末尾に追加することです。この関数はバイトの要素をバイトのスライスに追加し、必要であればスライスを拡張します、そして更新されたスライス値を返します:

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

このように `AppendByte` を使うことができます:

```
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

`AppendByte` のような関数はスライスの拡張を完璧に制御するので便利です。プログラムの性質に合わせて、チャンクをより大きくまたはより小さくアロケーションしたり、再アロケーションするサイズの上限を増やしたりするのが望ましいかもしれません。

しかしほとんどのプログラムは完璧に制御する必要がないので、Goはほとんどの目的に合うビルトインの `append` 関数を提供しています。それはシグネチャを持っています。

```
func append(s []T, x ...T) []T 
```

`append` 関数は要素 `x` をスライス `s` の末尾に追加し、より大きな容量が必要であればスライスを拡張します。

```
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

あるスライスを別のスライスに追加する場合は、第2引数を引数のリストに伸長するために `...` を使います。

```
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

スライスのゼロ値（`nil`）は長さ0のスライスのように振る舞うので、あなたはスライス変数を宣言した後ループ内でそれに追加できます:

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

## "理解" できる

最初に述べた通り、あるスライスの再スライシングでは元の配列のコピーを作りません。満杯の配列は参照されなくなるまでメモリに残り続けるでしょう。これでは小さなデータの断片が欲しいときだけメモリ上に全てのデータを保持するプログラムをときどき生み出してしまいます。

例えば、この `FindDigits` 関数はメモリにファイルを読み込み、連続する数字列のうち最初の組を探し、それを新しいスライスとして返します。

```
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

このコードは宣言した通りに振る舞いますが、戻り値 `[]byte` はファイル全体を含む配列を指しています。スライスは元の配列を参照するため、スライスはガベージコレクタが配列を解放できない状態にずっとします。ファイルのほんの一部を利用するだけだったのにファイルの中身全体をメモリ上に保持しています。

この問題を解決するために、注目しているデータを返す前に新しいスライスにコピーします:

```
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

この関数のより簡潔なバージョンは `append` を使って作ることができます。これは読者への練習問題として残しておきます。

## 関連記事

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