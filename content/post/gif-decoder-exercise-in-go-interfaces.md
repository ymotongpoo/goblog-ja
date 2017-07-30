+++
date = "2011-05-25T00:07:38+09:00"
title = "GIFデコーダ: Goインターフェースの練習（A GIF decoder: an exercise in Go interfaces）"
draft = false
tags = ["gif", "gopher", "image", "interface", "lagomorph", "lzw", "moustache", "rodent", "technical"]
+++

# GIFデコーダ: Goインターフェースの練習
[A GIF decoder: an exercise in Go interfaces](https://blog.golang.org/gif-decoder-exercise-in-go-interfaces) by Rob Pike

## はじめに

2011年5月10日にサンフランシスコで行われたGoogle I/Oのカンファレンスで、私たちはGo言語がGoogle App Engineで利用可能になったことを発表しました。Goは機械語に直接コンパイルするApp Engine上で利用可能となった最初の言語で、画像処理のようなCPUに負荷をかけるタスクにとってそれは良い選択でした。

その流れで、私たちは以下のような画像を手軽により良くする [Moustachio](http://moustach-io.appspot.com/) と呼ばれるプログラムを実演しました:

![gif-decoder-exercise-in-go-interfaces_image00](./gif-decoder-exercise-in-go-interfaces_image00.jpg)

髭を加えて、その結果を共有しましょう:

![gif-decoder-exercise-in-go-interfaces_image02](./gif-decoder-exercise-in-go-interfaces_image02.jpg)

アンチエイリアスが施された髭の描画を含む全てのグラフィック処理は、App Engine上で動作しているGoプログラムで完結します。（そのソースコードは [appengine-goプロジェクト](http://code.google.com/p/appengine-go/source/browse/example/moustachio/) で利用可能です。）

Web上のほとんどの画像 - 少なくとも髭加工される可能性のある - がJPEGであるにも関わらず、ほかにも数えきれないほど広まっている画像形式があり、そしてアップロードされた数種類の画像形式を受け入れるのでそれは髭にとってもの分かりが良いように見えます。JPEGとPNGのデコーダはすでにGoの画像ライブラリの中にありましたが、昔からあるGIF形式のデコーダはなかったので、私たちは発表に間に合うようにGIFデコーダを書くとこにしました。そのデコーダは、問題解決のためにGoのインターフェースがどのようにしてその問題をより扱いやすくしているのかを示すいくつかのピースを含んでいます。このブログ記事の残りの部分ではその2、3個の例を述べています。

## GIFのフォーマット

まず初めに、GIFの形式について簡単に見ていきましょう。GIFの画像ファイルは*パレット化*されており、つまりそれぞれのピクセル値はファイルに含まれているある決まったカラーマップにインデックス付けされています。ディスプレイの1ピクセルがたった8ビットで表されていた頃からGIF形式はあり、カラーマップは値の制限された組をスクリーンを明るくするために必要なRGB（赤、緑、青）の3値に変換するために使われていました。（これはJPEGとは対称的で、例えば、JPEGはエンコーダが直接カラー信号の分離を表現するためJPEGにはカラーマップはありません。）

GIF画像は1ピクセル当たり1から8ビットの値をとることができ、包括的ですが、1ピクセルあたり8ビットが最も使われています。

少し単純化すると、GIFファイルはピクセル深度、画像の次元、カラーマップ（1枚の8ビット画像あたり256色のRGB値）をそれぞれ定義するヘッダーと、次いでピクセルデータを含んでいます。ピクセルデータは1次元のビットストリームとして格納され、写真には向きませんがコンピュータが生成するグラフィックスにとってはかなり効率的なLZWアルゴリズムを使って圧縮されます。そのとき圧縮データはある長さで区切られ、1バイトのカウント（0-255）とそれに続くバイト列という構成のブロックに分割されます:

![gif-decoder-exercise-in-go-interfaces_image03](./gif-decoder-exercise-in-go-interfaces_image03.gif)

## ピクセルデータのデブロッキング

GIFのピクセルデータをGoでデコードするために、`compress/lzw` パッケージからLZWデコンプレッサを使うことができます。そのパッケージには、[ドキュメント](http://golang.org/pkg/compress/lzw/#NewReader)曰く、「rから読み出したデータを解凍することによって読み出し可能となる」オブジェクトを返すNewReader関数があります。

```
func NewReader(r io.Reader, order Order, litWidth int) io.ReadCloser
````

ここで、`order`はビットデータをパックする順番を定義し、`litWidth`はGIFファイルとピクセル深度（典型的には8）を対応させる際に使用するビット単位のワードサイズを意味しています。

しかし、`NewReader` の最初の引数として入力ファイルを与えることはできません。それはデコンプレッサがバイトストリームを要求しているにも関わらずGIFデータはアンパックが必要なブロックストリームになっているからです。この問題を扱うために、それをデブロッキングするためのちょっとしたコードにより入力 `io.Reader` をラップすることができます。さらにそのコードを再び `Reader` として実装することもできます。つまり、デブロッキングするコードを `blockReader` と呼ばれる新しい型の `Read` メソッドの中で実装します。

以下は `blockReader` のデータ構造です。

```
type blockReader struct {
    r     reader    // Input source; implements io.Reader and io.ByteReader.
    slice []byte    // Buffer of unread data.
    tmp   [256]byte // Storage for slice.
}
```

リーダー `r` は画像データのソースであり、そのソースは恐らくファイルかHTTP接続でしょう。`slice` と `tmp` フィールドはデブロッキングを管理するために使われます。以下は `Read` メソッドの全体像です。

```
1  func (b *blockReader) Read(p []byte) (int, os.Error) {
2      if len(p) == 0 {
3          return 0, nil
4      }
5      if len(b.slice) == 0 {
6          blockLen, err := b.r.ReadByte()
7          if err != nil {
8              return 0, err
9          }
10         if blockLen == 0 {
11             return 0, os.EOF
12         }
13         b.slice = b.tmp[0:blockLen]
14         if _, err = io.ReadFull(b.r, b.slice); err != nil {
15             return 0, err
16         }
17     }
18     n := copy(p, b.slice)
19     b.slice = b.slice[n:]
20     return n, nil
21 }
```

2-4行目はちょうどサニティーチェック（異常がないかの確認）に当たります。データを置くところがなければ0を返します。起こり得ないですが、念のためです。

5行目は `b.slice` の長さを確認することによって前回の呼び出しから左側にデータがあるかどうかを訊ねています。もしなければ、スライスの長さは0であり、`r` から次のブロックを読み出す必要があります。

GIFブロックは1バイトのカウントで始まり、6行目で読み出しています。もしカウントが0であれば、GIFは、最後のブロックになったためにこれを定義します。したがって `EOF` を11行目で返します。

今、私たちは `blockLen` バイト読むべきだと分かっています。ですので、`b.tmp` の最初の `blockLen` バイトが `b.slice` を指すようにし、たくさんのバイトデータ読み出すためにヘルパー関数 `io.ReadFull` を使います。

18-19行目は `b.slice` から呼び出し側（訳注：レシーバー `b` のこと）のバッファに対しデータをコピーします。私たちは `Read` を実装していますが、`ReadFull` は実装していないので、要求されたバイト数よりも少ないバイト数を返すことを許可されています。それを実装するのは簡単です: `b.slice` から呼び出し側のバッファ（`p`）にデータをコピーし、コピー関数からの戻り値がコピーされたバイト数です。そのとき最初の `n` バイトを切り落とすために `b.slice` をリサイズし、次の呼び出しに備えています。

スライス（`b.slice`）を配列（`b.tmp`）に結びつけて考えるのはGoプログラミングにおいては良いテクニックです。この場合、`blockReader` 型の `Read` メソッドは決して全てをアロケーションしないということを意味しています。カウント周り（スライス長に暗に示されています）を管理する必要がないことも意味しており、ビルトインの `copy` 関数がこれ以上コピーしないことを保証しています。（スライスについてより詳しく知りたい場合は、[the Go Blog のこの記事](http://blog.golang.org/2011/01/go-slices-usage-and-internals.html)をご覧ください。）

`blockReader` 型が与えられると、画像データストリームを入力リーダーをラップすることによりアンパックできます。ファイルについてはこのようになります:

```
deblockingReader := &blockReader{r: imageFile}
```

このラッピングはブロックに区切られたGIF画像ストリームを `blockReader` の `Read` メソッドを呼び出すことにより利用可能な単純なバイトストリームに変換します。

## ピースを繋げる

`blockReader` の実装とライブラリから利用可能なLZWコンプレッサにより、画像データストリームをデコードするために必要なピースが全て揃いました。驚くほど短く、素直にコードを紡ぎ合わせます。

```
lzwr := lzw.NewReader(&blockReader{r: d.r}, lzw.LSB, int(litWidth))
if _, err = io.ReadFull(lzwr, m.Pix); err != nil {
   break
}
```

以上です。

最初の行は `blockReader` を作り、デコンプレッサを作るためにそれを `lzw.NewReader` に通します。ここで、`d.r` は画像データを保持する `io.Reader` で、`lzw.LSB` はLZWデコンプレッサ内でのバイトオーダーを定義し、`litWidth` はピクセル深度です。

解析器が与えられると、次の行ではデータを解凍するための `io.ReadFull` を呼び出し、それを画像 `m.Pix` に格納しています。`ReadFull` から戻ると、画像データは解凍され表示可能な画像 `m` に格納されます。

このコードはまず初めに動作します。本当です。

`NewReader` 呼び出しをちょうど `blockReader` を `NewReader` 呼び出しの中で組み立てたように、`ReadFull` の引数リストに置くことで一時変数 `lzwr` を避けることができますが、1行のコードに詰め過ぎかもしれません。

## 結論

データを再構築するためにこのように部品を組み立てることにより、Goのインターフェースはソフトウェアを組み立てやすくします。この例では、型安全なUnixパイプラインのように私たちはデブロッカーと `io.Reader` インターフェースを用いたデコンプレッサを一緒に繋ぎ合わせGIFデコーダを実装しました。さらに私たちは暗に示された `Reader` インターフェースの実装としてデブロッカーも書き、そのときはパイプライン処理に合うような外部宣言や雛形を必要としませんでした。ほとんどの言語においてとてもコンパクトに、それでもなお綺麗にそして安全にこのデコーダを実装するのは難しいですが、そのインターフェース機構はGoでほとんど自然にそれを作る約束事を減らすことに対しプラスとなります。

それは他の写真（この場合GIF）に対しても価値があります:

![gif-decoder-exercise-in-go-interfaces_image01](./gif-decoder-exercise-in-go-interfaces_image01.gif)

GIFの形式は [http://www.w3.org/Graphics/GIF/spec-gif89a.txt](http://www.w3.org/Graphics/GIF/spec-gif89a.txt) で定義されています。

*By Rob Pike*

## あわせて読みたい

* [HTTP/2 Server Push](https://blog.golang.org/h2push)
* [Introducing HTTP Tracing](https://blog.golang.org/http-tracing)
* [GopherCon 2015 Roundup](https://blog.golang.org/gophercon2015)
* [Generating code](https://blog.golang.org/generate)
* [The Go Gopher](https://blog.golang.org/gopher)
* [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
* [Go maps in action](https://blog.golang.org/go-maps-in-action)
* [go fmt your code](https://blog.golang.org/go-fmt-your-code)
* [Organizing Go code](https://blog.golang.org/organizing-go-code)
* [The Go Programming Language turns two](https://blog.golang.org/go-programming-language-turns-two)
* [Debugging Go programs with the GNU Debugger](https://blog.golang.org/debugging-go-programs-with-gnu-debugger)
* [The Go image/draw package](https://blog.golang.org/go-imagedraw-package)
* [The Go image package](https://blog.golang.org/go-image-package)
* [The Laws of Reflection](https://blog.golang.org/laws-of-reflection)
* [Error handling and Go](https://blog.golang.org/error-handling-and-go)
* ["First Class Functions in Go"](https://blog.golang.org/first-class-functions-in-go-and-new-go)
* [Profiling Go Programs](https://blog.golang.org/profiling-go-programs)
* [Go at Google I/O 2011: videos](https://blog.golang.org/go-at-google-io-2011-videos)
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
