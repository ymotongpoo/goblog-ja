+++
date = "2014-07-29T09:00:00+09:00"
draft = false
title = "Goの並行パターン：コンテキスト (Go Concurrency Pattern: Context)"
+++

# Goの並行パターン：コンテキスト
[Go Concurrency Pattern: Context](https://blog.golang.org/context) by Sameer Ajmani

## はじめに
Goで書かれたサーバでは、サーバに来たリクエストはそれぞれそれ自身のゴルーチンで処理されます。
リクエストハンドラはしばしばデータベースやRPCサービスといったバックエンドにアクセスするために追加でゴルーチンを起動します。
リクエストの処理を行っているゴルーチンは、通常エンドユーザのアイデンティティや認可トークン、リクエストの期限などリクエスト固有の値へのアクセス権が必要です。
リクエストがキャンセルされたりタイムアウトした場合には、システムがそれらのゴルーチンが使っていたリソースを再度要求することができるように、
そのリクエストの処理を行っていたすべてのゴルーチンは素早く終了すべきです。

Googleで私たちは、簡単にAPIの境界をまたぐリクエスト固有の値やキャンセルのシグナル、期限などを、
あるリクエストに関係するすべてのゴルーチンに投げることが出来る、 `context` パッケージというパッケージを開発しました。
パッケージは [golang.org/x/net/context](http://godoc.org/golang.org/x/net/context) に公開されています。 [1][]
この記事ではそのパッケージの使い方と実際に動作する例を紹介したいと思います。

  [1] 訳註: 原文では `code.google.com/p/go.net/context` を参照していますが、現状に合わせてURLを変更しました。

## コンテキスト（Context）

`context` パッケージの核となっているのは `Context` 型です。

```
// ContextはAPIの境界を越えて期限とキャンセルシグナルとリクエスト固有の値を保持します。
// メソッドは複数のゴルーチンから同時に呼び出されても安全です。
type Context interface {
    // Doneはこのコンテキストがキャンセルされたりタイムアウトした場合にcloseされます。
    Done() <-chan struct{}

    // ErrはDoneチャンネルが閉じた後なぜこのコンテキストがキャンセルされたかを知らせます。
    Err() error

    // Deadlineは設定されている場合にはいつこのContextがキャンセルされるかを返します。
    Deadline() (deadline time.Time, ok bool)

    // Valueはkeyに紐付いた値を返し、設定がない場合はnilを返します。
    Value(key interface{}) interface{}
}
```

（この説明は要約されたもので、 [godoc](http://godoc.org/golang.org/x/net/context) が正式なものです。）

`Done` メソッドは、 `Context` の代わりに動作する関数に対するキャンセルシグナルとして
振る舞うチャンネルを返します。チャンネルが閉じられたときに、関数は処理を中断して戻るべきです。
`Err` メソッドはなぜその `Context` がキャンセルされたかを示すエラーを返します。
[パイプラインとキャンセル](https://blog.golang.org/pipelines) の記事では `Done` チャンネルの
イディオムについてより詳細に議論しています。

`Context` には `Done` チャンネルが受信専用であるのと同様の理由で `Cancel` メソッドがあり _ません_ 。
キャンセルシグナルを受け取る関数は通常シグナルを送る関数ではありません。
特に、親の操作が子の操作を行うゴルーチンを起動したときに、これら子の操作は親をキャンセルできるべきではありません。
代わりに `WithCancel` 関数（あとで説明します）で新しい `Context` の値をキャンセルする方法を提供します。

`Context` は複数のゴルーチンから同時に使われても安全です。コード内では1つの `Context` を
任意の数のゴルーチンに渡し、その `Context` をキャンセルしてすべてのゴルーチンに伝えることができます。

`Deadline` は関数が処理を始めるべきかどうかを決定することができるメソッドです。
もし残り時間が少なければ、起動する価値はありません。コードでは期限をI/O操作のタイムアウトとして利用することもあるでしょう。

`Value` によって `Context` がリクエスト固有のデータを運ぶことができます。
そのようなデータは複数のゴルーチンによって同時に利用されても安全です。

## 派生したコンテキスト

`context` パッケージでは既存のコンテキストから新しい `Context` の値を _派生する_ 関数を提供しています。
これらの値は木構造になっていて、 `Context` がキャンセルされたときに、そこから派生した `Context` もすべてキャンセルされます。


`Background` はすべての `Context` 木構造の根になっていて、決してキャンセルされることはありません。

```
// Background は空の Context を返します。
// そのコンテキストは決してキャンセルされることはなく、期限はなく、また値を持ちません。
// Backgroundは通常、main、init、テストの中で使われ、到着するリクエストの
// 最上位の Context として使われます。
func Background() Context
```

`WithCancel` と `WithTimeout` は派生した `Context` を返します。
この `Context` は親の `Context` よりも早くキャンセルされます。
到着するリクエストに紐付いた `Context` は通常リクエストハンドラが戻すとキャンセルされます。
`WithCancel` は複数のレプリカを使うときにまた冗長なリクエストをキャンセルするのにも便利です。
`WithTimeout` はバックエンドサーバーへのリクエストの期限を設定するのに便利です。

```
// WithCancel は parent のコピーを返し、 parent.Done が閉じられた、
// またはキャンセルが呼ばれるとすぐに、その Done チャンネルが閉じられます。
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// CancelFunc は Context をキャンセルします。
type CancelFunc func()

// WithTimeout は parent のコピーを返し、 parent.Done が閉じられた、
// または timeout が過ぎるとすぐに、その Done チャンネルが閉じられます。
// Context の Deadline は 現在時刻+timeout か親の期限のどちらか早いほうに設定されます。
// もしタイマーがまだ動いていた場合、キャンセル関数はそのリソースを解放します。
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithValue` はリクエスト固有の値を `Context` に紐付ける方法を提供します。

```
// WithValue は親のコピーを返し、そのValueメソッドがkeyに対しvalを返すようにします。
func WithValue(parent Context, key interface{}, val interface{}) Context
```

`context` パッケージの使い方を理解するには動く実例を通して見るのが最良でしょう。

## 例: Google ウェブ検索
一つの例は `/search?q=golang&timeout=1s` のようなURLを処理して「golang」という検索クエリを
[Google Web Search API](https://developers.google.com/web-search/docs/) に投げて、
その結果を表示するようなHTTPサーバーです。 `timeout` パラメータはサーバーに指定時間が経過したら
リクエストをキャンセルするように伝えます。

コードは3つのパッケージに分かれています。

* [server](https://blog.golang.org/context/server/server.go) は `main` 関数と `/search` のハンドラーを提供します。
* [userip](https://blog.golang.org/context/userip/userip.go) はリクエストからユーザーのIPアドレスを抜き出し、 `Context` に紐付ける関数を提供します。

* [google](https://blog.golang.org/context/google/google.go) はGoogleにクエリを送信する `Search` 関数を提供します。

## サーバーのプログラム
[server](https://blog.golang.org/context/server/server.go) のプログラムは `/serach?q=golang` のようなリクエストを処理して
`golang` という検索クエリによるGoogle検索の最初の結果いくつかを返します。サーバーでは `handleSearch` という関数を `/search` のエンドポイントとして
登録しています。ハンドラーは `ctx` という最初の `Context` を生成して、ハンドラーが値を返すときにそれがキャンセルされるように設定します。
もしリクエストに `timeout` というURLパラメーターが含まれていたら、 `Context` は自動的に期限が来たらキャンセルされます。

```
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx はこのハンドラーの Context です。 cancel を呼ぶことで
    // ctx.Done チャンネルが閉じられます。これで、このハンドラーからのリクエスト用の
    // キャンセルシグナルです。
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // リクエストにはタイムアウトがあるので、期限が来たら自動的にキャンセルされる
        // コンテキストを作成します。
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // handleSearchが値を返したらすぐに ctx をキャンセルします。
```

このハンドラーは `google.Search` を `ctx` と `query` を使って呼び出します。

```
    // Google検索を実行して結果を表示します。
    start := time.Now()
    results, err := google.Search(ctx, query)
    elapsed := time.Since(start)
```

検索に成功したら、ハンドラーは結果を返します。

```
    if err := resultsTemplate.Execute(w, struct {
        Results          google.Results
        Timeout, Elapsed time.Duration
    }{
        Results: results,
        Timeout: timeout,
        Elapsed: elapsed,
    }); err != nil {
        log.Print(err)
        return
    }
```
