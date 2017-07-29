+++
date = "2011-09-21T22:46:24+09:00"
draft = false
title = "Go image パッケージ（The Go image package）"
tags = ["image", "libraries", "technical"]
+++

# Go image パッケージ

[The Go image package](https://blog.golang.org/go-image-package) By Nigel Tao

## はじめに

[image](http://golang.org/pkg/image/) と [image/color](http://golang.org/pkg/image/color/) パッケージはいくつかの型を定義しています。`color.Color` と `color.Model` は色を、`image.Point` と `image.Rectangle` は基本的な2次元幾何学をそれぞれ記述しており、`image.Image` は色の長方形格子を表現するためにそれら2つの概念を1つにまとめます。[別記事](http://golang.org/doc/articles/image_draw.html) では [image/draw](http://golang.org/pkg/image/draw/) を用いた画像の構成について取り上げています。

## Color と Color Model

[Color](http://golang.org/pkg/image/color/#Color) は色として考えられる任意の型をまとめた最小限のメソッドの組を定義したインターフェースです。つまり、赤、緑、青、アルファ値に変換できるものです。CMYK や YCbCr 色空間から変換するといったある種の変換は不可逆かもしれません。

```
type Color interface {
    // RGBA returns the alpha-premultiplied red, green, blue and alpha values
    // for the color. Each value ranges within [0, 0xFFFF], but is represented
    // by a uint32 so that multiplying by a blend factor up to 0xFFFF will not
    // overflow.
    RGBA() (r, g, b, a uint32)
}

戻り値には3つの重要な巧妙さがあります。1つ目は、赤、緑、青はプリマルチプライド・アルファであることです。25% 透明な完全飽和赤は 75% の r を返す RGBA によって表現されます。2つ目に、チャネルの有効範囲は 16 ビットです。CMYK や YCbCr からの変換は不可逆なので、100% の赤は 255 ではなく 65535 の r を返す RGBA によって表現されています。3つ目に、最大値は 65535 ですが、戻り値の型は `uint32` となっていることです。これは2つの値の積がオーバーフローしないことを保証するためです。[Porter-Duff の](https://en.wikipedia.org/wiki/Alpha_compositing)古典代数学のスタイルでアルファマスクに従う2つの色を混ぜ合わせて第3の色を生成する乗算際に起ります。

```
dstr, dstg, dstb, dsta := dst.RGBA()
srcr, srcg, srcb, srca := src.RGBA()
_, _, _, m := mask.RGBA()
const M = 1<<16 - 1
// The resultant red value is a blend of dstr and srcr, and ranges in [0, M].
// The calculation for green, blue and alpha is similar.
dstr = (dstr*(M-m) + srcr*m) / M
```

もしノン・プリマルチプライド・アルファを使って処理を行おうとすると、コード片の最後の行はより複雑になるでしょう。それが `Color` がプリマルチプライド・アルファな値を使う理由です。

image/color パッケージも `Color` インターフェースを実装するいくつかの完全な型を定義しています。例えば、[`RGBA`](http://golang.org/pkg/image/color/#RGBA) は古典的な "1 チャネルあたり 8 ビット" の色を表現する構造になっています。

```
type RGBA struct {
    R, G, B, A uint8
}
```

`RGBA` の `R` フィールドは 0 から 255 の範囲の値をとる 8 ビットのプリマルチプライド・アルファな色であることに注意してください。0 から 65535 の範囲の値をとる 16 ビットのプリマルチプライド・アルファな色を生成するために 0x101 を値に掛けることによって `RGBA` は `Color` インターフェースを満足させます。同様に、PNG の画像形式で使われるように [`NRGBA`](http://golang.org/pkg/image/color/#NRGBA) 構造型は 8 ビットのノン・プリマルチプライド・アルファな色を表現します。

[`Model`](http://golang.org/pkg/image/color/#Model) は単純なもので、おそらく非可逆的に `色` を別の `色` に変換することができます。例えば、`GrayModel` は 任意の `色` を不飽和化された [`Gray`](http://golang.org/pkg/image/color/#Gray) に変換できます。 `Palette` は制限されたパレットにより任意の `色` をある `色` に変換できます。

```
type Model interface {
    Convert(c Color) Color
}

type Palette []Color
```

## Point と Rectangle

[`Point`](http://golang.org/pkg/image/#Point) は右方向と下方向に増加する軸に従った整数格子上の (x, y) 座標です。それはピクセルでも格子で区切られた正方形でもありません。`点` は固有の幅や高さ、色を持っていませんが、以下の図では色付きの小さな正方形を用いています。

```
type Point struct {
    X, Y int
}
```

![go-image-package_image-package-01](./go-image-package_image-package-01.png)

```
p := image.Point{2, 1}
```

[`Rectangle`](http://golang.org/pkg/image/#Rectangle) は左上と右下の `点` によって定義される整数格子上にある軸に平行な長方形です。`長方形` も固有の色を持っていませんが、以下の図では色付きの細い線を用いて長方形の輪郭を描き、`最小` および `最大` の `点` を呼びます。

```
type Rectangle struct {
    Min, Max Point
}
```

便利のため `image.Rect(x0, y0, x1, y1)` は `image.Rectangle{image.Point{x0, y0}, image.Point{x1, y1}}` と同値としますが、前者のほうが入力するのは簡単です。

`長方形` は左上の点を排他し、右下の点を包含しています。`点 p` と `長方形 r` に対し、`r.Min.X <= p.X && p.X < r.Max.X` を満たす場合のみ `p.In(r)` と表します、`Y` についても同様です。これはスライス `s[i0:i1]` がどうやって下限を含み上限を排他するのかということに似ています。（配列やスライスと違い、`長方形` はよく非ゼロの原点を含みます。）

![go-image-package_image-package-02](./go-image-package_image-package-02.png)

```
r := image.Rect(2, 1, 5, 5)
// Dx and Dy return a rectangle's width and height.
fmt.Println(r.Dx(), r.Dy(), image.Pt(0, 0).In(r)) // prints 3 4 false
```

`点` を `長方形` に加えることでその `長方形` を変換します。点と長方形は右下の象限内に制限されていません。

![go-image-package_image-package-03](./go-image-package_image-package-03.png)

```
r := image.Rect(2, 1, 5, 5).Add(image.Pt(-4, -2))
fmt.Println(r.Dx(), r.Dy(), image.Pt(0, 0).In(r)) // prints 3 4 true
```

2つの長方形の興味深いところは、もう1つの長方形を生み出すところです、それは空かもしれません。

![go-image-package_image-package-04](./go-image-package_image-package-04.png)

```
r := image.Rect(0, 0, 4, 3).Intersect(image.Rect(2, 2, 5, 5))
// Size returns a rectangle's width and height, as a Point.
fmt.Printf("%#v\n", r.Size()) // prints image.Point{X:2, Y:1}
```

点と長方形は値によってパスされたりリターンされたりします。`Rectangle` を引数に取る関数は2つの `Point` か4つの `int` を引数にとる関数と同じくらい優れているでしょう。

## Image

[Image](http://golang.org/pkg/image/#Image) は `長方形` 内の格子で区切られた正方形を `モデル` から作られる `色` に射影します。"(x, y) のピクセル" は点 (x, y), (x+1, y), (x+1, y+1), (x, y+1) によって定義される格子で区切られた正方形の色を参照します。

```
type Image interface {
    // ColorModel returns the Image's color model.
    ColorModel() color.Model
    // Bounds returns the domain for which At can return non-zero color.
    // The bounds do not necessarily contain the point (0, 0).
    Bounds() Rectangle
    // At returns the color of the pixel at (x, y).
    // At(Bounds().Min.X, Bounds().Min.Y) returns the upper-left pixel of the grid.
    // At(Bounds().Max.X-1, Bounds().Max.Y-1) returns the lower-right one.
    At(x, y int) color.Color
}
```

共通する誤りは `画像` の範囲が (0, 0) から始まると思い込むことです。例えば、アニメーション GIF が画像列を含み、典型的に初めの画像以降のそれぞれの `画像` が変化した領域のピクセルデータだけ保持し、その領域は (0, 0) から始まる必要はありません。`画像` m のピクセル全体を走査するための正しい方法は以下のような感じです:

```
b := m.Bounds()
for y := b.Min.Y; y < b.Max.Y; y++ {
    for x := b.Min.X; x < b.Max.X; x++ {
        doStuffWith(m.At(x, y))
    }
}
```

`画像` の実装はピクセルデータのインメモリスライスをベースにする必要はありません。例えば、[`Uniform`](http://golang.org/pkg/image/#Uniform) は巨大でインメモリ表現が単なる一様な色の `画像` です。

```
type Uniform struct {
    C color.Color
}
```

典型的に、それでもなお、プログラムはスライスベースの画像を求めています。[`RGBA`](http://golang.org/pkg/image/#RGBA) や [`Gray`](http://golang.org/pkg/image/#Gray) のような構造型（他のパッケージでは `image.RGBA` や `image.Gray` として参照する）はピクセルデータのスライスを保持し `Image` インターフェースを実装します。

```
type RGBA struct {
    // Pix holds the image's pixels, in R, G, B, A order. The pixel at
    // (x, y) starts at Pix[(y-Rect.Min.Y)*Stride + (x-Rect.Min.X)*4].
    Pix []uint8
    // Stride is the Pix stride (in bytes) between vertically adjacent pixels.
    Stride int
    // Rect is the image's bounds.
    Rect Rectangle
}
```

それらの型も一度に1ピクセルを修正する `Set(x, y int, c color.Color)` メソッドを提供します。

```
m := image.NewRGBA(image.Rect(0, 0, 640, 480))
m.Set(5, 5, color.RGBA{255, 0, 0, 255})
```

たくさんのピクセルデータを読み書きしたい場合はより効率的にできますが、構造型の `Pix` フィールドに直接アクセスするためより複雑になります。

スライスベースの `画像` の実装も同じ配列を元にした `画像` を返す `SubImage` メソッドを提供しています。サブスライス `s[i0:i1]` の内容を修正すると元のスライス `s` の内容に影響が及ぶことと同様に、サブ画像のピクセルを修正すると元の画像のピクセルに影響が及びます。

![go-image-package_image-package-05](./go-image-package_image-package-05.png)

```
m0 := image.NewRGBA(image.Rect(0, 0, 8, 5))
m1 := m0.SubImage(image.Rect(1, 2, 5, 5)).(*image.RGBA)
fmt.Println(m0.Bounds().Dx(), m1.Bounds().Dx()) // prints 8, 4
fmt.Println(m0.Stride == m1.Stride)             // prints true
```

画像の `Pix` フィールドで動作する低級なコードを扱ううえで、`Pix` の範囲を超えることは画像の範囲外のピクセルに影響を与えるということを理解しておいてください。上記の例では、`m1.Pix` によって変換されたピクセルは青で共有されます。`At` や `Set` メソッドまたは [image/draw パッケージ](http://golang.org/pkg/image/draw/) のような高級なコードは画像の範囲に対する操作を制限します。

## 画像の形式

標準パッケージライブラリは GIF、JPEG、PNG などの多くの共通した画像形式をサポートしています。もとの画像ファイルのフォーマットが分かっている場合、[`io.Reader`](http://golang.org/pkg/io/#Reader) から直接デコードできます。

```
import (
    "image/jpeg"
    "image/png"
    "io"
)

// convertJPEGToPNG converts from JPEG to PNG.
func convertJPEGToPNG(w io.Writer, r io.Reader) error {
    img, err := jpeg.Decode(r)
    if err != nil {
        return err
    }
    return png.Encode(w, img)
}
```

形式不明な画像データがある場合、[`image.Decode`](http://golang.org/pkg/image/#Decode) 関数は形式を検出することができます。認識される形式の集合は実行時に構築され、標準パッケージライブラリ内の形式に制限されていません。画像形式パッケージは典型的に init 関数内で自身を登録し、main パッケージは形式の登録が副作用するようにそのようなパッケージ単体を "アンダースコアインポート" するでしょう。

```
import (
    "image"
    "image/png"
    "io"

    _ "code.google.com/p/vp8-go/webp"
    _ "image/jpeg"
)

// convertToPNG converts from any recognized format to PNG.
func convertToPNG(w io.Writer, r io.Reader) error {
    img, _, err := image.Decode(r)
    if err != nil {
        return err
    }
    return png.Encode(w, img)
}
```

*By Nigel Tao*

## あわせて読む
* [HTTP/2 Server Push](https://blog.golang.org/h2push)
* [Introducing HTTP Tracing](https://blog.golang.org/http-tracing)
* [Generating code](https://blog.golang.org/generate)
* [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
* [Go maps in action](https://blog.golang.org/go-maps-in-action)
* [go fmt your code](https://blog.golang.org/go-fmt-your-code)
* [Organizing Go code](https://blog.golang.org/organizing-go-code)
* [Debugging Go programs with the GNU Debugger](https://blog.golang.org/debugging-go-programs-with-gnu-debugger)
* [The Go image/draw package](https://blog.golang.org/go-imagedraw-package)
* [The Laws of Reflection](https://blog.golang.org/laws-of-reflection)
* [Error handling and Go](https://blog.golang.org/error-handling-and-go)
* ["First Class Functions in Go"](https://blog.golang.org/first-class-functions-in-go-and-new-go)
* [Profiling Go Programs](https://blog.golang.org/profiling-go-programs)
* [Spotlight on external Go libraries](https://blog.golang.org/spotlight-on-external-go-libraries)
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