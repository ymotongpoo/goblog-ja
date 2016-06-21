+++
date = "2013-09-26T09:42:44+09:00"
draft = true
title = "配列、スライス（と文字列）：'append'の動作原理 (Arrays, slices (and strings): The mechanics of 'append')"
tags = ["array", "slice", "string", "copy", "append"]
+++

# 配列、スライス（と文字列）：'append'の動作原理
[Arrays, slices (and strings): The mechanics of 'append'](https://blog.golang.org/slices) By Rob Pike

## はじめに

手続き型プログラミング言語において共通した機能のひとつに配列という概念があります。配列は単純に見えて、言語に追加する際には
答えなければならない多くの設問にこたえなければいけません。たとえば次のようなものです。

* 固定長なのか、可変長なのか
* 長さは型に含めるのか
* 多次元配列はどのように表現するか
* 空の配列が何を意味するか

これらの設問に対する回答が、配列が単に言語の一機能になるか、それとも言語設計の中心になるかをわけます。

Goの開発の初期段階で、言語設計がしっくりくるまで、これらの設問に対する答えを決めるのに1年かかりました。　
鍵となったのはスライスの導入でした。スライスによって、固定長の配列に柔軟性や拡張性をもったデータ構造を与えました。
しかしながら、今日まで、Goを使い始めたばかりのプログラマはスライスの動作の理解にしばしばつまづいています。
おそらくそれは他の言語での経験がスライスに対する考え方に影響しているからでしょう。

この記事では、その混乱を解決しようと思います。そのために、ひとつひとつ部品を積み重ねて、組み込み関数の `append` がどのように動作するか、
そしてなぜそのように動作するのかを説明します。

## 配列

Goにおいて、配列は重要な要素ですが、工事における基礎のように、配列はより見えやすい構成要素の下に隠されています。
より面白くて、強力で、輝くアイデアであるスライスの話をする前に、まずは簡単に配列の説明をしましょう。

配列はGoのプログラム内にはあまり見られません。なぜなら配列の大きさは型の一部で、それによって表現力が制限されるからです。

次の宣言では

```
var buffer [256]byte
```

は変数バッファを宣言しています。このバッファは256バイトを保持します。 `buffer` は型その大きさをを型情報に含んでいて
`[256]byte` となっています。512バイトの配列は、その専用の型である `[512]byte` 型となります。

配列に紐付いたデータは、たった一つ、配列の要素だけです。構文の観点でいえば、さきほどの `buffer` はメモリ上では次のようになっています。

```
buffer: byte byte byte ... 256 times ... byte byte byte
```

つまり、変数は256バイトのデータを保持していて、ただそれだけです。私たちは各要素によく知られたインデックスの構文でアクセスできます。
`buffer[0]` 、 `buffer[1]` から始まって `buffer[255]` までです。（0から255までインデックスの範囲で256の要素をカバーしています）
この範囲の外側にある値のインデックスにアクセスしようとすると、プログラムがクラッシュします。

`len` という名前の組み込み関数があり、これは配列、スライス、また他のいくつかのデータ型の要素数を返します。
配列においては、 `len` が何を返すかは明らかです。私たちの例では `len(buffer)` は固定値の `256` を返します。

配列には使われるべき場所があります。インスタンスの変換行列として良い表現になっています。しかし、Goで配列がもっともよく使われるのは、
スライス内部で保存領域を確保する目的で使われるときです。

## スライス: スライスのヘッダー

スライスでは面白い処理が行われていますが、スライスを上手に使うためには、それがどのような性質を持っていて、どんな振る舞いをするかを
きちんと理解しなければいけません。

スライスはスライスの変数自身とは別に保存された配列の境界を表現したデータ構造です。スライスは配列の一部を表現しています。

前の説の `buffer` という配列の変数を再利用して、100番目の要素から150番目の要素（正確には含まれる要素は100から149）を持つスライスを、
配列から切り取る（スライスする）ことで作ることもできるでしょう。

```
var slice []byte = buffer[100:150]
```

上のスニペットでは、明示的になるように変数宣言を冗長な形で書きました。変数 `slice` は `[]byte` 型で、「バイトのスライス」と呼びます。
`slice` は `buffer` という名前の配列から、100（含む）から150（除く）の要素を切り取ることで初期化されました。
よりイディオムに近い構文では、初期化表現で追加される型情報を省略します。

```
var slice = buffer[100:150]
```

関数内では、短い記法を使うこともできます。

```
slice := buffer[100:150]
```

この `slice` 変数とは一体なんでしょう。すべてを説明するわけではありませんが、とりあえずスライスを長さとポインターという2つの要素を持つ
小さなデータ構造とみなしてみましょう。するとスライスが作られるときには、その背後で次のようなことが起きていると考えられます。

```
type sliceHeader struct {
    Length        int
    ZerothElement *byte
}

slice := sliceHeader{
    Length:        50,
    ZerothElement: &buffer[100],
}
```

もちろん、これは単なる説明にすぎません。このスニペットが `sliceHeader` 構造体はプログラマに見えないことを説明していて、
要素のポインターの型が要素の型に依存するものとは言え、このスニペットは一般的な仕組みを説明しています。

これまで、配列を切り取る操作をしてきましたが、スライスを切り取ることもできます。次のとおりです。

```
slice2 := slice[5:10]
```

配列から切り取ったときと同様に、この操作では新しいスライスを作成します。この場合は元のスライスの要素5から9（含む）のスライスで、
これはつまり元の配列の要素105から109を意味します。スライスの変数 `slice2` の中の配列が基にしている `sliceHeader` 構造体は次のようになります。

```
slice2 := sliceHeader{
    Length:        5,
    ZerothElement: &buffer[105],
}
```

このヘッダーが依然として `buffer` 変数の中にあるものと同じ配列を参照していることに注意して下さい。

再切り取り（再スライス）、つまりスライスを切り取って、その結果を同じスライスに保存することもできます。

```
slice = slice[5:10]
```

この操作のあとは、 `slice` 変数の `sliceHeader` 構造体は `slice2` 変数のものと同様になります。
たとえばスライスと切り詰めるときに再切り取りをよく目にします。次の式では `slice` の最初と最後の要素を捨てます。

```
slice = slice[1:len(slice)-1]
```

（演習：この代入のあとに `sliceHeader` 構造体がどのようになるかを書いてみてください）

経験あるGoプログラマがしばしば「スライスヘッダー」について話すのを聞くでしょう。なぜなら、スライスヘッダーは本当にスライス変数内に
保存されているからです。たとえば、 `bytes.IndexRune` のようなスライスを引数に取る関数を呼び出すとき、そのヘッダーが
関数に渡される実態です。この関数呼び出しでは

```
slashPos := bytes.IndexRune(slice, '/')
```

`IndexRune` 関数に渡される `slice` 引数は、実際には「スライスヘッダー」なのです。 

スライスヘッダー内にはもう一つのデータ要素があり、それは以降で説明しますが、まずはプログラムでスライスを使うときに、
このスライスヘッダーが存在することが何を意味するのかを理解しましょう。

## スライスを関数に渡す

スライスがポインターを持っていたとしても、スライスそれ自身は値であることを理解することが重要です。
一皮めくれば、スライスはポインターと長さを持つ構造体の値です。構造体のポインターではありません。

この事実が大事です。

先の例で `IndexRune` を呼んだとき、 `IndexRune` にはスライスヘッダーのコピーが渡されました。
この振る舞いは重要な結果をもたらします。

次の簡単な関数を考えてみましょう。

```
func AddOneToEachElement(slice []byte) {
    for i := range slice {
        slice[i]++
    }
}
```

これは関数名が示す通り、スライスの各インデックスを（`for range` ループを使って）繰り返し進めていき、各要素を1増加させています。

試してみましょう。

```
func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }
    fmt.Println("before", slice)
    AddOneToEachElement(slice)
    fmt.Println("after", slice)
}
```

実行結果は次の通り。

```
before [0 1 2 3 4 5 6 7 8 9]
after [1 2 3 4 5 6 7 8 9 10]
```

スライスヘッダーは値で渡されたとしても、配列の要素に対するポインターを含んでいるため、元のスライスヘッダーと関数に渡された
スライスヘッダーのコピーは同じ配列を表しています。それゆえ、関数がreturnするとき、変更された要素は元の `slice` 変数からでも
確認できるのです。

関数に渡された引数は本当にコピーです。次の例で確認できます。

```
func SubtractOneFromLength(slice []byte) []byte {
    slice = slice[0 : len(slice)-1]
    return slice
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    newSlice := SubtractOneFromLength(slice)
    fmt.Println("After:  len(slice) =", len(slice))
    fmt.Println("After:  len(newSlice) =", len(newSlice))
}
```

実行結果：

```
Before: len(slice) = 50
After:  len(slice) = 50
After:  len(newSlice) = 49
```

スライスの引数の中身は関数によって修正可能だけど、そのヘッダーは修正不可能であることを確認しました。
`slice` 変数に保持されている長さは関数の呼び出しでは変更されません。なぜなら関数にはスライスヘッダーのコピーが渡されていて、
元のスライスヘッダーが渡されているわけではないからです。したがって、もしヘッダーを変更する関数を書きたいのであれば、
いま例示したように、それを結果として返さなければいけません。 `slice` 変数は変更されていませんが、返ってきた値は新しい長さになっていて、
それが `newSlice` に保存されます。

## スライスへのポインター：メソッドレシーバー

スライスヘッダーを変更する関数を書く場合、関数にスライスヘッダーのポインターを渡すというのも手です。
先ほどの例の違うバージョンを書いてみます。

```
func PtrSubtractOneFromLength(slicePtr *[]byte) {
    slice := *slicePtr
    *slicePtr = slice[0 : len(slice)-1]
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    PtrSubtractOneFromLength(&slice)
    fmt.Println("After:  len(slice) =", len(slice))
}
```

実行結果:

```
Before: len(slice) = 50
After:  len(slice) = 49
```

この例では、特に間接的な代入をしているところ（一時変数を使っています）がぎこちなく見えますが、ポインターへのスライスではよくある例です。
スライスを変更するメソッドにポインターレシーバーを使うイディオムがあります。

スライスに最後のスラッシュ以降を捨てるメソッドを持たせたいとしましょう。そのメソッドは次のように書けます。

```
type path []byte

func (p *path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(*p, []byte("/"))
    if i >= 0 {
        *p = (*p)[0:i]
    }
}

func main() {
    pathName := path("/usr/bin/tso") // string型からpath型への変換
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)
}
```

このサンプルを実行してみると、呼び出し元のスライスを期待通り更新していることがわかると重います。

実行結果:

```
/usr/bin
```

（演習：レシーバーの型をポインターではなく値に描き変えて再度実行してみましょう。実行結果がなぜそうなるか説明して下さい。）

一方で、（非英語のパス名は無視していますが）パス内のASCII文字を大文字に変換するメソッドを書きたい場合、
メソッドは値レシーバーでも構いません。なぜなら値レシーバーでも同じ配列を参照しているからです。

```
type path []byte

func (p path) ToUpper() {
    for i, b := range p {
        if 'a' <= b && b <= 'z' {
            p[i] = b + 'A' - 'a'
        }
    }
}

func main() {
    pathName := path("/usr/bin/tso")
    pathName.ToUpper()
    fmt.Printf("%s\n", pathName)
}
```

実行結果:

```
/USR/BIN/TSO
```

ここで `ToUpper` メソッドはインデックスとスライスの要素を取得するために `for range` 構文の中で2つの変数を使っています。
この形によって、`for` の中身で `p[i]` を何度も書くことを回避しています。

（演習： `ToUpper` メソッドをポインターレシーバーに変更して、動作が変わるか確認しましょう）
（応用演習： `ToUpper` メソッドでASCII文字だけではなくUnicode文字を扱えるようにしてみましょう）

## 容量

次の、intのスライスの引数を1要素だけ拡大する関数を見てみましょう。

```
func Extend(slice []int, element int) []int {
    n := len(slice)
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

（なぜ修正したスライスを返す必要があるのでしょうか）実行してみます。

```
func main() {
    var iBuffer [10]int
    slice := iBuffer[0:0]
    for i := 0; i < 20; i++ {
        slice = Extend(slice, i)
        fmt.Println(slice)
    }
}
```

実行結果:

```
[0]
[0 1]
[0 1 2]
[0 1 2 3]
[0 1 2 3 4]
[0 1 2 3 4 5]
[0 1 2 3 4 5 6]
[0 1 2 3 4 5 6 7]
[0 1 2 3 4 5 6 7 8]
[0 1 2 3 4 5 6 7 8 9]
panic: runtime error: slice bounds out of range

goroutine 1 [running]:
panic(0x13f420, 0x1040a018)
	/usr/local/go/src/runtime/panic.go:481 +0x700
main.main()
	/tmp/sandbox346230702/main.go:27 +0x220
```

スライスの容量が20まで増えているのがおわかりに・・・増えていません。

スライスヘッダーの3つ目の要素についてお話するときがやってきました。スライスの容量です。
スライスヘッダーには、配列へのポインターと長さに加えて、スライスの容量も保持しています。

```
type sliceHeader struct {
    Length        int
    Capacity      int
    ZerothElement *byte
}
```

`Capacity` フィールドは内部で持っている配列に実際どれくらいの空き容量があるかを記録しています。
この値は `Length` が取りうる最大です。スライスをその容量を超えて拡大させることは配列の制限を超えることであり、
パニックを引き起こす原因となります。

先の例で `slice` を作成したあと

```
slice := iBuffer[0:0]
```

ヘッダーは次のようになっています。

```
slice := sliceHeader{
    Length:        0,
    Capacity:      10,
    ZerothElement: &iBuffer[0],
}
```

`Capacity` フィールドは、内部で持っている配列の長さからスライスの最初の要素の配列におけるインデックス（この場合は0）を引いたものとなります。
スライスの容量を知りたければ、組み込み関数の `cap` を使います。

```
if cap(slice) == len(slice) {
    fmt.Println("slice is full!")
}
```

## make

スライスをその容量以上に大きくしたい時にはどうしたらいいのでしょうか。出来ません！
スライスの定義では、容量が大きくできる最大値です。しかし、求めている結果と同じことが、新しい配列を確保して、データをそこにコピーして、
スライスが新しい配列を指すように変更することで可能となります。

まず新しい配列を確保するところからはじめましょう。より大きな配列を確保して結果を切り取るために組み込み関数の `new` を使うこともできますが、
代わりに組み込み関数の `make` を使うほうが簡潔にできます。 `make` は新しい配列を確保して、それを指すスライスヘッダーを作成するということを
一度に行います。 `make` 関数は3つの引数を取ります。スライスの型、初期値の長さと容量です。容量はスライスのデータを保持するために `make` が
確保する配列の長さを意味します。次の関数呼び出しで、長さ10で余裕が5（15-10）あるスライスを作成しています。

```
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

実行結果:

```
len: 10, cap: 15
```

このスニペットではintのスライスの容量を倍にし、長さは同じままに保っています。

```
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
    newSlice := make([]int, len(slice), 2*cap(slice))
    for i := range slice {
        newSlice[i] = slice[i]
    }
    slice = newSlice
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

このコードを実行すると、スライスは、配列を再確保する必要ができるまでに、ずっと多くの余裕ができます。

実行結果:

```
len: 10, cap: 15
len: 10, cap: 30
```

スライスを作成するとき、長さと容量が同じであることがしばしばあります。組み込み関数の `make` にはこのよくある状況に合わせた
短い書き方があります。長さの引数を容量のデフォルト値にすることができます。容量の引数を書かなければ、長さと容量を同じ値にできます。

次のコードを実行すると

```
gophers := make([]Gopher, 10)
```

`gophers` スライスは長さと容量が10に設定されます。

## copy

前の説でスライス `slice` の容量を倍にしたときに、古いデータを新しいスライスにコピーするためにループを書きました。
この処理をより簡単にするために、Goには `copy` という組み込み関数があります。引数は2つのスライスで、右側の引数から左側の引数に
データをコピーします。 `copy` を使って前の例を書き直すと次のようになります。

```
    newSlice := make([]int, len(slice), 2*cap(slice))
    copy(newSlice, slice)
```

実行結果:

```
len: 10, cap: 15
len: 10, cap: 30
```

`copy` 関数は頭が良いです。両方の引数の長さに注意を払いながら、コピー可能な分だけコピーをします。
言い換えれば、`copy` 関数がコピーする要素の数は、2つの引数の長さの短い方です。
要素の大小を自分で管理しなくて済みます。また、常に確認する必要はないですが、 `copy` はコピーした要素の数を整数値で返します。

`copy` 関数はコピー元とコピー先に重なるところがある場合にもきちんと動作します。つまり、1つのスライス内で要素を動かしたい場合です。
次の例では `copy` を使ってどのようにスライスの真ん中に値を挿入するかをお見せします。

```
// Insertは値をスライス中の指定したインデックスに挿入します。
// 指定するインデックスはスライスの範囲内でなければいけません。
// スライスには新しい要素が入る余地がなければいけません。
func Insert(slice []int, index, value int) []int {
    // スライスを1要素だけ大きくする
    slice = slice[0 : len(slice)+1]
    // 指定したインデックスよりも大きいインデックスをずらして1要素分隙間をあける
    copy(slice[index+1:], slice[index:])
    // 新しい値を保存する
    slice[index] = value
    // 結果を返す
    return slice
}
```

この関数で注意することがいくつかあります。もちろん、まずはじめに、スライスの長さが変わったので、更新されたスライスを返さなければいけません。
つぎに、この例では便利な略記法を使っています。この式は

```
slice[i:]
```

つぎの式とまったく同じ意味です。

```
slice[i:len(slice)]
```

また、まだこの使い方は見せていませんが、スライス記法で最初の要素を省略することも可能です。デフォルトでは0になります。
したがって

```
slice[:]
```

はそのスライス自身を表すことにほかなりません。この書き方は配列をスライスにするときに便利です。
この書き方は「ある配列の要素をすべて持つスライス」を宣言するのに最も短く書ける方法です。

```
array[:]
```

話がそれてしまいました。 `Insert` 関数を実行してみましょう。

```
    slice := make([]int, 10, 20) // capacity > length であることに注目。要素を追加するための余地です。
    for i := range slice {
        slice[i] = i
    }
    fmt.Println(slice)
    slice = Insert(slice, 5, 99)
    fmt.Println(slice)
```

実行結果:

```
[0 1 2 3 4 5 6 7 8 9]
[0 1 2 3 4 99 5 6 7 8 9]
```

## `Append`: 例

2,3前の節で、スライスを1要素分だけ拡張する関数 `Extend` を書きました。しかしながら、あの実装はバグがありました。
スライスの容量が小さすぎる場合、関数がクラッシュしてしまうからです。（`Insert` の例も同様の問題があります。）
いまは修正するための部品がそろったので、intのスライスに対して堅牢な実装の `Extend` を書いてみましょう。

```
func Extend(slice []int, element int) []int {
    n := len(slice)
    if n == cap(slice) {
        // スライスがいっぱいなとき。拡大しなければいけない。
        // サイズが0でも拡大できるように2倍+1のサイズにする。
        newSlice := make([]int, len(slice), 2*len(slice)+1)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

この場合、スライスを返すという部分が特に重要です。理由は、結果のスライスを再確保したとき、それはまったく異なる配列を参照しているからです。
スライスがいっぱいになったらどうなるか、短いスニペットで確かめてみましょう。

```
    slice := make([]int, 0, 5)
    for i := 0; i < 10; i++ {
        slice = Extend(slice, i)
        fmt.Printf("len=%d cap=%d slice=%v\n", len(slice), cap(slice), slice)
        fmt.Println("address of 0th element:", &slice[0])
    }
```

実行結果:

```
len=1 cap=5 slice=[0]
address of 0th element: 0x10430200
len=2 cap=5 slice=[0 1]
address of 0th element: 0x10430200
len=3 cap=5 slice=[0 1 2]
address of 0th element: 0x10430200
len=4 cap=5 slice=[0 1 2 3]
address of 0th element: 0x10430200
len=5 cap=5 slice=[0 1 2 3 4]
address of 0th element: 0x10430200
len=6 cap=11 slice=[0 1 2 3 4 5]
address of 0th element: 0x10436120
len=7 cap=11 slice=[0 1 2 3 4 5 6]
address of 0th element: 0x10436120
len=8 cap=11 slice=[0 1 2 3 4 5 6 7]
address of 0th element: 0x10436120
len=9 cap=11 slice=[0 1 2 3 4 5 6 7 8]
address of 0th element: 0x10436120
len=10 cap=11 slice=[0 1 2 3 4 5 6 7 8 9]
address of 0th element: 0x10436120
```

最初の大きさ5の配列がいっぱいになったときに再確保していることに注目して下さい。
新しい配列が確保されたときに、容量の0番目の要素のアドレスが変わっています。

堅牢な `Extend` 関数を参考にして、複数の要素でスライスを拡張できるずっと便利な関数を書くことが出来ます。
この実装では、Goで関数が呼びだされてた時に、関数の引数をスライスに変換できる機能を使います。
つまりGoでの関数の可変数引数の機能を使います。

いまから書く関数を `Append` としましょう。最初のバージョンでは、可変数引数の部分を明確にできるよう、
単純に `Extend` を繰り返し呼び出します。 `Append` のシグネチャーは次のようになります。

```
func Append(slice []int, items ...int) []int
```

これは `Append` は1つの引数 `slice` を取り、それ以降に0以上のintの変数が続くことを意味しています。
これらの引数は、見て分かる通り、 `Append` がintを扱う限り、intのスライスです。

```
// Appendはスライスにitemsを追加します。
// 最初のバージョン：ただExtendを繰り返し呼び出す。
func Append(slice []int, items ...int) []int {
    for _, item := range items {
        slice = Extend(slice, item)
    }
    return slice
}
```

`for range` ループが引数 `items` の要素に対して繰り返し `Extend` を呼び出していてます。
`items` の暗黙的な型は `[]int` です。またブランク識別子 `_` を使って、ループのインデックスを捨てていることも注目してください。
今回の実装ではインデックスは必要ありません。

試してみます。

```
    slice := []int{0, 1, 2, 3, 4}
    fmt.Println(slice)
    slice = Append(slice, 5, 6, 7, 8)
    fmt.Println(slice)
```

実行結果:

```
[0 1 2 3 4]
[0 1 2 3 4 5 6 7 8]
```

この例で新しいテクニックが消化されています。スライスをリテラルを書いて初期化しています。書き方は、スライスの型につづいて、
各要素をブレースで囲むというものです。

```
    slice := []int{0, 1, 2, 3, 4}
```

`Append` 関数は他の理由でも興味深いです。要素を追加するだけではなく、呼び出し時に `...` 記法を使って
「展開」させることで、2つめのスライスをまるごと追加することが出来ます。

```
    slice1 := []int{0, 1, 2, 3, 4}
    slice2 := []int{55, 66, 77}
    fmt.Println(slice1)
    slice1 = Append(slice1, slice2...) // The '...' is essential!
    fmt.Println(slice1)
```

実行結果:

```
[0 1 2 3 4]
[0 1 2 3 4 55 66 77]
```

もちろん、確保を1度だけしか行わず、`Extend`の中でスライスを作るようにすることで `Append` をより効率的にすることができます。 

```
// Appendはスライスに要素を追加します。
// 効率的なバージョン。
func Append(slice []int, elements ...int) []int {
    n := len(slice)
    total := len(slice) + len(elements)
    if total > cap(slice) {
        // 再確保。新しいサイズは1.5倍なので、さらに拡大することができます。
        // Reallocate. Grow to 1.5 times the new size, so we can still grow.
        newSize := total*3/2 + 1
        newSlice := make([]int, total, newSize)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[:total]
    copy(slice[n:], elements)
    return slice
}
```

ここで、`copy` を2度使っているその使い方に注目して下さい。1回めはスライスのデータを新しく確保したメモリに移動するため、
2回めは追加する要素を古いデータのあとにコピーするために使っています。

試してみましょう。動作は前の例と同様です。

```
    slice1 := []int{0, 1, 2, 3, 4}
    slice2 := []int{55, 66, 77}
    fmt.Println(slice1)
    slice1 = Append(slice1, slice2...) // The '...' is essential!
    fmt.Println(slice1)
```

実行結果:

```
[0 1 2 3 4]
[0 1 2 3 4 55 66 77]
```

## `append`: 組み込み関数

ようやく組み込み関数の `append` を設計する動機までたどり着きました。この関数は先の `Append` の例と同じ処理を行い、
同じくらい効率的で、それでいてあらゆるスライス型で動作します。

Goの弱点として、ジェネリクス型の操作はランタイムに行わなければいけないというものがあります。
いつか変わるかもしれませんが、いまは、スライスの操作が簡単になるように、汎用の `append` 関数を提供しています。
機能は先のintのスライスのバージョンと同じですが、あらゆるスライス型に対応している点が異なります。

覚えておいてほしいのは、スライスヘッダーは `append` を呼び出すたびに更新されるので、
呼び出したあとは返されたスライスを保存しなければいけません。事実、`append` を呼んだあとに結果を保存しないと
コンパイラがエラーを吐きます。

次のコードは `append` を使った言い回しをprint文とともに並べたものです。
そのまま実行したり、編集したりしながら、機能を確認してみてください。

```
    // 2つのスライスを作成します。
    slice := []int{1, 2, 3}
    slice2 := []int{55, 66, 77}
    fmt.Println("Start slice: ", slice)
    fmt.Println("Start slice2:", slice2)

    // スライスにアイテムを追加します。
    slice = append(slice, 4)
    fmt.Println("Add one item:", slice)

    // スライスを他のスライスに追加します。
    slice = append(slice, slice2...)
    fmt.Println("Add one slice:", slice)

    // （intの）スライスのコピーを作ります。
    slice3 := append([]int(nil), slice...)
    fmt.Println("Copy a slice:", slice3)

    // スライスを自分自身の後ろにコピーします。
    fmt.Println("Before append to self:", slice)
    slice = append(slice, slice...)
    fmt.Println("After append to self:", slice)
```

実行結果:

```
Start slice:  [1 2 3]
Start slice2: [55 66 77]
Add one item: [1 2 3 4]
Add one slice: [1 2 3 4 55 66 77]
Copy a slice: [1 2 3 4 55 66 77]
Before append to self: [1 2 3 4 55 66 77]
After append to self: [1 2 3 4 55 66 77 1 2 3 4 55 66 77]
```

スライスの設計によって簡潔な記述できちんと動作することを理解できるので、時間を取って最後の例の詳細な動作を考えてみましょう。

 `append` や `copy` などのスライスの操作の例は、コミュニティが作成した ["Slice Tricks" というWikiページ](https://golang.org/wiki/SliceTricks) にあります。

## Nil

余談として、これまで得た知識を考えると、 `nil` スライスがどのように表現されるかわかります。
自然に考えるとスライスヘッダーのゼロ値は

```
sliceHeader{
    Length:        0,
    Capacity:      0,
    ZerothElement: nil,
}
```

あるいは

```
sliceHeader{}
```

と表現されます。重要な点は、要素のポインターも `nil` になっている点です。次の配列から作られたスライスは

```
array[0:0]
```

長さ0（かつおそらく容量0）ですが、そのポインターは `nil` ではないので、これは `nil` スライスではありません。

明確にすると、空のスライスは（容量が0ではないと想定すると）拡大することができますが、 `nil` スライスは値を入れる配列がなく、
拡大して1つの要素も保持することさえできません。

つまり、`nil` スライスは、たとえポインターが何も指していないとしても、機能的には長さ0のスライスと同値です。
長さ0のスライスは、長さが0で確保することで要素を追加することが出来ます。
例として、先の例でスライスを `nil` スライスに追加しているものを確認して下さい。

## 文字列 (string)

この節では、スライスという観点から、Goにおける文字列の扱いについて簡単に説明します。

文字列は、実際に非常に単純です。文字列は読み込み専用のバイトのスライスにいくつか言語からの構文サポートがついたものです。　

読み込み専用なので、（拡大させることができないことから）容量を考える必要がありません。かわりにたいていの目的においては、
単なる読み込み専用のバイトのスライスとして扱うことが出来ます。

まずはじめに、個々のバイトにインデックスでアクセスすることが出来ます。

```
slash := "/usr/ken"[0] // '/' のバイト値を返します。
```

部分文字列を取得するために文字列を切り取ることが出来ます。

```
usr := "/usr/ken"[0:4] // "/usr" という文字列を返します。
```

文字列を切り取るときに、その裏で何が起きているかは明らかでしょう。

単なるバイトのスライスから、単純にキャストすることで文字列を作ることも出来ます。

```
str := string(slice)
```

同様に逆も可能です。

```
slice := []byte(usr)
```

文字列の内部にある配列は見えないようになっていて、文字列（string）を通してしかその配列の中身にはアクセスできません。
つまり、文字列とバイトスライスの間で変換を行うと、かならず配列のコピーが発生するということです。
もちろん、Goがコピー処理を行うので、自分で行う必要はありません。文字列からバイトスライスに変換した際には、
変換後のバイトスライス内の配列を操作しても、変換元の文字列には影響しません。

文字列をスライスのような設計にしたことによる重要な帰結として、部分文字列を非常に効率よく精製できるようになったことがあります。
部分文字列を作るときに必要なことは、2つのフィールドを持つ文字列のヘッダーを生成することです。
文字列は読み取り専用なので、元の文字列とそこから得られた部分文字列は同じ配列を安全に共有できます。

歴史的経緯：最初期の文字列の実装では常に再確保していましたが、Go言語にスライスがついかされたときに、スライスによって
効率的な文字列操作のためのモデルが提供されました。結果としていくつかのベンチマークで大幅な高速化が確認されました。

もちろん、文字列に関してはもっと語ることがありますが、それは[別のブログポスト](./string)でより深く説明することにします。

## Conclusion

To understand how slices work, it helps to understand how they are implemented. There is a little data structure, the slice header, that is the item associated with the slice variable, and that header describes a section of a separately allocated array. When we pass slice values around, the header gets copied but the array it points to is always shared.

Once you appreciate how they work, slices become not only easy to use, but powerful and expressive, especially with the help of the copy and append built-in functions.

## More reading

There's lots to find around the intertubes about slices in Go. As mentioned earlier, the ["Slice Tricks" Wiki page](https://golang.org/wiki/SliceTricks) has many examples. The [Go Slices](http://blog.golang.org/go-slices-usage-and-internals) blog post describes the memory layout details with clear diagrams. Russ Cox's [Go Data Structures](http://research.swtch.com/godata) article includes a discussion of slices along with some of Go's other internal data structures.

There is much more material available, but the best way to learn about slices is to use them.

By Rob Pike

## あわせて読みたい
* [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)åå
