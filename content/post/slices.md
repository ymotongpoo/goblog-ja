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

## Capacity

Look at the following function that extends its argument slice of ints by one element:

```
func Extend(slice []int, element int) []int {
    n := len(slice)
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

(Why does it need to return the modified slice?) Now run it:

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

See how the slice grows until... it doesn't.

It's time to talk about the third component of the slice header: its capacity. Besides the array pointer and length, the slice header also stores its capacity:

```
type sliceHeader struct {
    Length        int
    Capacity      int
    ZerothElement *byte
}
```

The Capacity field records how much space the underlying array actually has; it is the maximum value the Length can reach. Trying to grow the slice beyond its capacity will step beyond the limits of the array and will trigger a panic.

After our example slice is created by

```
slice := iBuffer[0:0]
```

its header looks like this:

```
slice := sliceHeader{
    Length:        0,
    Capacity:      10,
    ZerothElement: &iBuffer[0],
}
```

The Capacity field is equal to the length of the underlying array, minus the index in the array of the first element of the slice (zero in this case). If you want to inquire what the capacity is for a slice, use the built-in function cap:

```
if cap(slice) == len(slice) {
    fmt.Println("slice is full!")
}
```

## Make

What if we want to grow the slice beyond its capacity? You can't! By definition, the capacity is the limit to growth. But you can achieve an equivalent result by allocating a new array, copying the data over, and modifying the slice to describe the new array.

Let's start with allocation. We could use the new built-in function to allocate a bigger array and then slice the result, but it is simpler to use the make built-in function instead. It allocates a new array and creates a slice header to describe it, all at once. The make function takes three arguments: the type of the slice, its initial length, and its capacity, which is the length of the array that make allocates to hold the slice data. This call creates a slice of length 10 with room for 5 more (15-10), as you can see by running it:

```
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

実行結果:

```
len: 10, cap: 15
```

This snippet doubles the capacity of our int slice but keeps its length the same:

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

実行結果:

```
len: 10, cap: 15
len: 10, cap: 30
```

After running this code the slice has much more room to grow before needing another reallocation.

When creating slices, it's often true that the length and capacity will be same. The make built-in has a shorthand for this common case. The length argument defaults to the capacity, so you can leave it out to set them both to the same value. After

```
gophers := make([]Gopher, 10)
```

the gophers slice has both its length and capacity set to 10.

## Copy

When we doubled the capacity of our slice in the previous section, we wrote a loop to copy the old data to the new slice. Go has a built-in function, copy, to make this easier. Its arguments are two slices, and it copies the data from the right-hand argument to the left-hand argument. Here's our example rewritten to use copy:

```
    newSlice := make([]int, len(slice), 2*cap(slice))
    copy(newSlice, slice)
```

実行結果:

```
len: 10, cap: 15
len: 10, cap: 30
```

The copy function is smart. It only copies what it can, paying attention to the lengths of both arguments. In other words, the number of elements it copies is the minimum of the lengths of the two slices. This can save a little bookkeeping. Also, copy returns an integer value, the number of elements it copied, although it's not always worth checking.

The copy function also gets things right when source and destination overlap, which means it can be used to shift items around in a single slice. Here's how to use copy to insert a value into the middle of a slice.

```
// Insert inserts the value into the slice at the specified index,
// which must be in range.
// The slice must have room for the new element.
func Insert(slice []int, index, value int) []int {
    // Grow the slice by one element.
    slice = slice[0 : len(slice)+1]
    // Use copy to move the upper part of the slice out of the way and open a hole.
    copy(slice[index+1:], slice[index:])
    // Store the new value.
    slice[index] = value
    // Return the result.
    return slice
}
```

There are a couple of things to notice in this function. First, of course, it must return the updated slice because its length has changed. Second, it uses a convenient shorthand. The expression

```
slice[i:]
```

means exactly the same as

```
slice[i:len(slice)]
```

Also, although we haven't used the trick yet, we can leave out the first element of a slice expression too; it defaults to zero. Thus

```
slice[:]
```

just means the slice itself, which is useful when slicing an array. This expression is the shortest way to say "a slice describing all the elements of the array":

```
array[:]
```

Now that's out of the way, let's run our Insert function.

```
    slice := make([]int, 10, 20) // Note capacity > length: room to add element.
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

## Append: An example

A few sections back, we wrote an Extend function that extends a slice by one element. It was buggy, though, because if the slice's capacity was too small, the function would crash. (Our Insert example has the same problem.) Now we have the pieces in place to fix that, so let's write a robust implementation of Extend for integer slices.

```
func Extend(slice []int, element int) []int {
    n := len(slice)
    if n == cap(slice) {
        // Slice is full; must grow.
        // We double its size and add 1, so if the size is zero we still grow.
        newSlice := make([]int, len(slice), 2*len(slice)+1)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

In this case it's especially important to return the slice, since when it reallocates the resulting slice describes a completely different array. Here's a little snippet to demonstrate what happens as the slice fills up:

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

Notice the reallocation when the initial array of size 5 is filled up. Both the capacity and the address of the zeroth element change when the new array is allocated.

With the robust Extend function as a guide we can write an even nicer function that lets us extend the slice by multiple elements. To do this, we use Go's ability to turn a list of function arguments into a slice when the function is called. That is, we use Go's variadic function facility.

Let's call the function Append. For the first version, we can just call Extend repeatedly so the mechanism of the variadic function is clear. The signature of Append is this:

```
func Append(slice []int, items ...int) []int
```

What that says is that Append takes one argument, a slice, followed by zero or more int arguments. Those arguments are exactly a slice of int as far as the implementation of Append is concerned, as you can see:

```
// Append appends the items to the slice.
// First version: just loop calling Extend.
func Append(slice []int, items ...int) []int {
    for _, item := range items {
        slice = Extend(slice, item)
    }
    return slice
}
```

Notice the for range loop iterating over the elements of the items argument, which has implied type []int. Also notice the use of the blank identifier _ to discard the index in the loop, which we don't need in this case.

Try it:

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

Another new technique is in this example is that we initialize the slice by writing a composite literal, which consists of the type of the slice followed by its elements in braces:

```
    slice := []int{0, 1, 2, 3, 4}
```

The Append function is interesting for another reason. Not only can we append elements, we can append a whole second slice by "exploding" the slice into arguments using the ... notation at the call site:

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

Of course, we can make Append more efficient by allocating no more than once, building on the innards of Extend:

```
// Append appends the elements to the slice.
// Efficient version.
func Append(slice []int, elements ...int) []int {
    n := len(slice)
    total := len(slice) + len(elements)
    if total > cap(slice) {
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

Here, notice how we use copy twice, once to move the slice data to the newly allocated memory, and then to copy the appending items to the end of the old data.

Try it; the behavior is the same as before:

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

## Append: The built-in function

And so we arrive at the motivation for the design of the append built-in function. It does exactly what our Append example does, with equivalent efficiency, but it works for any slice type.

A weakness of Go is that any generic-type operations must be provided by the run-time. Some day that may change, but for now, to make working with slices easier, Go provides a built-in generic append function. It works the same as our int slice version, but for any slice type.

Remember, since the slice header is always updated by a call to append, you need to save the returned slice after the call. In fact, the compiler won't let you call append without saving the result.

Here are some one-liners intermingled with print statements. Try them, edit them and explore:

```
    // Create a couple of starter slices.
    slice := []int{1, 2, 3}
    slice2 := []int{55, 66, 77}
    fmt.Println("Start slice: ", slice)
    fmt.Println("Start slice2:", slice2)

    // Add an item to a slice.
    slice = append(slice, 4)
    fmt.Println("Add one item:", slice)

    // Add one slice to another.
    slice = append(slice, slice2...)
    fmt.Println("Add one slice:", slice)

    // Make a copy of a slice (of int).
    slice3 := append([]int(nil), slice...)
    fmt.Println("Copy a slice:", slice3)

    // Copy a slice to the end of itself.
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

It's worth taking a moment to think about the final one-liner of that example in detail to understand how the design of slices makes it possible for this simple call to work correctly.

There are lots more examples of append, copy, and other ways to use slices on the community-built ["Slice Tricks" Wiki page](https://golang.org/wiki/SliceTricks).

## Nil

As an aside, with our newfound knowledge we can see what the representation of a nil slice is. Naturally, it is the zero value of the slice header:

```
sliceHeader{
    Length:        0,
    Capacity:      0,
    ZerothElement: nil,
}
```

or just

```
sliceHeader{}
```

The key detail is that the element pointer is nil too. The slice created by

```
array[0:0]
```

has length zero (and maybe even capacity zero) but its pointer is not nil, so it is not a nil slice.

As should be clear, an empty slice can grow (assuming it has non-zero capacity), but a nil slice has no array to put values in and can never grow to hold even one element.

That said, a nil slice is functionally equivalent to a zero-length slice, even though it points to nothing. It has length zero and can be appended to, with allocation. As an example, look at the one-liner above that copies a slice by appending to a nil slice.

## Strings

Now a brief section about strings in Go in the context of slices.

Strings are actually very simple: they are just read-only slices of bytes with a bit of extra syntactic support from the language.

Because they are read-only, there is no need for a capacity (you can't grow them), but otherwise for most purposes you can treat them just like read-only slices of bytes.

For starters, we can index them to access individual bytes:

```
slash := "/usr/ken"[0] // yields the byte value '/'.
```

We can slice a string to grab a substring:

```
usr := "/usr/ken"[0:4] // yields the string "/usr"
```

It should be obvious now what's going on behind the scenes when we slice a string.

We can also take a normal slice of bytes and create a string from it with the simple conversion:

```
str := string(slice)
```

and go in the reverse direction as well:

```
slice := []byte(usr)
```

The array underlying a string is hidden from view; there is no way to access its contents except through the string. That means that when we do either of these conversions, a copy of the array must be made. Go takes care of this, of course, so you don't have to. After either of these conversions, modifications to the array underlying the byte slice don't affect the corresponding string.

An important consequence of this slice-like design for strings is that creating a substring is very efficient. All that needs to happen is the creation of a two-word string header. Since the string is read-only, the original string and the string resulting from the slice operation can share the same array safely.

A historical note: The earliest implementation of strings always allocated, but when slices were added to the language, they provided a model for efficient string handling. Some of the benchmarks saw huge speedups as a result.

There's much more to strings, of course, and a [separate blog post](http://blog.golang.org/strings) covers them in greater depth.

## Conclusion

To understand how slices work, it helps to understand how they are implemented. There is a little data structure, the slice header, that is the item associated with the slice variable, and that header describes a section of a separately allocated array. When we pass slice values around, the header gets copied but the array it points to is always shared.

Once you appreciate how they work, slices become not only easy to use, but powerful and expressive, especially with the help of the copy and append built-in functions.

## More reading

There's lots to find around the intertubes about slices in Go. As mentioned earlier, the ["Slice Tricks" Wiki page](https://golang.org/wiki/SliceTricks) has many examples. The [Go Slices](http://blog.golang.org/go-slices-usage-and-internals) blog post describes the memory layout details with clear diagrams. Russ Cox's [Go Data Structures](http://research.swtch.com/godata) article includes a discussion of slices along with some of Go's other internal data structures.

There is much more material available, but the best way to learn about slices is to use them.

By Rob Pike

## あわせて読みたい
* [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)åå
