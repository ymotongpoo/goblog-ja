+++
date = "2014-07-29T09:00:00+09:00"
draft = false
title = "Goの並行パターン：コンテキスト (Go Concurrency Pattern: Context)"
tags = ["concurrency", "cancellation", "context", "gorilla", "tomb"]
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

## userip パケージ
[userip](https://blog.golang.org/context/userip/userip.go) パッケージはリクエストからユーザーのIPアドレスを抜き出し、
それを `Context` に紐付ける関数を提供します。 `Context` はキーと値の対応表を提供します。このとき、キーも値もともに `interface{}` 型です。
キーの型は同値性をサポートしなければならず、値の型は複数のゴルーチンから同時に使われても安全でなければなりません。
`userip` のようなパッケージはこの対応表の詳細を隠し、特定の `Context` の値にたいして強く型付けされたアクセスを提供します。

キーの衝突を避けるために、 `userip` ではエクスポートされていない型である `key` を定義し、この型の値をコンテキストのキーとして使います。

```
// key型は他のパッケージで定義されているコンテキストのキーと衝突しないよう、エクスポートされません。
type key int

// userIPkey はユーザーのIPアドレスのためのコンテキストのキーです。値がゼロの場合は任意の値となります。
// もしこのパッケージが他のコンテキストのキーであったなら、別の整数値になっていたことでしょう。
const userIPKey key = 0
```

`FromRequest` は `userIP` の値を `http.Request` から抜き出します。

```
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
    }
```

`NewContext` は与えられた `userIP` の値を持つ新しい `Context` を返します。

```
func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}
```

`FromContext` は `Context` から `userIP` を抜き出します。

```
func FromContext(ctx context.Context) (net.IP, bool) {
    // ctx.Value が nil を返す場合、 ctx はキーになる値を持っていません。
    // このとき net.IP 型のアサーションは ok=false を返します。
    userIP, ok := ctx.Value(userIPKey).(net.IP)
    return userIP, ok
}
```

## google パッケージ
[google.Search](https://blog.golang.org/context/google/google.go) 関数は [Google Web Search API](https://developers.google.com/web-search/docs/)
に対してHTTPリクエストを送り、JSONエンコードされた結果をパースします。この関数は `Context` のパラメーター `ctx` を受け取り、もし `ctx.Done` が
閉じられていたら、リクエストが実行中だったとしても、直ちに結果を返します。

Google Web Search APIのリクエストには、クエリパラメータとして検索クエリとユーザーのIPアドレスが含まれています。

```
func Search(ctx context.Context, query string) (Results, error) {
    // Google Search API へのリクエストの準備
    req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
    if err != nil {
        return nil, err
    }
    q := req.URL.Query()
    q.Set("q", query)

    // ctx にユーザーのIPアドレスが会った場合、それをサーバーに転送します。
    // Google API はサーバーが起動したリクエストとエンドユーザーのリクエストを区別するために
    // ユーザーのIPアドレスを使います。
    if userIP, ok := userip.FromContext(ctx); ok {
        q.Set("userip", userIP.String())
    }
    req.URL.RawQuery = q.Encode()
```

`Search` はHTTPリクエストの発行、およびリクエストまたはレスポンスが処理中に `ctx.Done` が
閉じられた場合のキャンセル処理のためにヘルパー関数である `httpDo` を使います。
`Search` はHTTPレスポンスを処理するために `httpDo` にクロージャーを渡します。

```
    var results Results
    err = httpDo(ctx, req, func(resp *http.Response, err error) error {
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        // JSON形式の検索結果をパースする。
        // https://developers.google.com/web-search/docs/#fonje
        var data struct {
            ResponseData struct {
                Results []struct {
                    TitleNoFormatting string
                    URL               string
                }
            }
        }
        if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
            return err
        }
        for _, res := range data.ResponseData.Results {
            results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
        }
        return nil
    })
    // httpDo は先ほど渡したクロージャーが結果を返すのを待つので、ここで結果を読み込んでも安全です。
    return results, err
```

`httpDo` 関数はHTTPリクエストを実行し、そのレスポンスを新しいゴルーチンで処理します。
`ctx.Done` が閉じられた場合にはゴルーチンが終了する前にリクエストをキャンセルします。

```
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
    // HTTPリクエストをゴルーチン内で実行し、レスポンスを f に渡す。
    tr := &http.Transport{}
    client := &http.Client{Transport: tr}
    c := make(chan error, 1)
    go func() { c <- f(client.Do(req)) }()
    select {
    case <-ctx.Done():
        tr.CancelRequest(req)
        <-c // f が値を返すのを待つ
        return ctx.Err()
    case err := <-c:
        return err
    }
}
```

## Context用のコードを適用する

多くのサーバーフレームワークがリクエスト固有の値を保持するためのパッケージと型を提供しています。
既存のフレームワークを使ったコードと `Context` パラメータを期待するコードの間の架け橋として
`Context` インターフェースの新しい実装を定義することができます。

たとえば、Gorillaの [github.com/gorilla/context](http://github.com/gorilla/context) パッケージは、
HTTPリクエストからキーと値のペアへの対応表を提供することで、ハンドラーがデータと受け取ったリクエストを
紐付けることができるようになっています。 [gorilla.go](https://blog.golang.org/context/gorilla/gorilla.go) では、
`Value` メソッドがGorillaパッケージ内の特定のHTTPリクエストに紐付いた値を返すような `Context` の実装を提供しています。

他のパッケージでは `Context` と似たキャンセルの仕組みをサポートしてきています。たとえば [Tomb](http://godoc.org/gopkg.in/tomb.v2)
では、 `Dying` チャンネルを閉じることでキャンセルシグナルを送る `Kill` メソッドを提供しています。
また `Tomb` は処理用のゴルーチンが終了するのを待つ、 `sync.WaitGroup` に似たメソッドも提供しています。
[tomb.go](https://blog.golang.org/context/tomb/tomb.go) では、親の `Context` がキャンセルされる、もしくは
与えられた `Tomb` が殺された場合にキャンセルされる `Context` の実装を提供しています。

## 結論
Googleでは、Goプログラマに受信および送信のリクエストの間の経路での呼び出しにおいて、
すべての関数で第1引数に `Context` パラメーターを渡すことを要求しています。
これによって、多くの異なるチームが開発したGoのコードがお互いに上手く動くようになっています。
`Context` によって期限とキャンセルを単純に制御できるようになり、またセキュリティ上の認証情報といった
致命的な値がGoのプログラム内を適切に通過することを確実にしています。

`Context` 上に構築したいサーバーフレームワークは、そのパッケージと `Context` パラメーターがあると期待される
パッケージ間の橋渡しをするような `Context` の実装を提供すべきでしょう。そうするとクライアントライブラリは
`Context` を呼び出しのコードから受け取るでしょう。リクエスト固有のデータとキャンセルに関する共通の
インターフェースを構築することで、 `Context` はパッケージ開発者がスケーラブルなサービスを作るとために
コードを共有することをより簡単にします。

By Sameer Ajmani

## あわせて読みたい

* [Go Concurrency Patterns: Pipelines and cancellation](http://blog.golang.org/pipelines)
  * [Goの並行パターン：コンテキスト (Go Concurrency Pattern: Context)](../context/)
* [http://blog.golang.org/race-detector](http://blog.golang.org/race-detector)
* [Advanced Go Concurrency Patterns](http://blog.golang.org/advanced-go-concurrency-patterns)
* [Concurrency is not parallelism](http://blog.golang.org/concurrency-is-not-parallelism)
* [Go videos from Google I/O 2012](http://blog.golang.org/go-videos-from-google-io-2012)
* [Go Concurrency Patterns: Timing out, moving on](http://blog.golang.org/go-concurrency-patterns-timing-out-and)
* [Share Memory By Communicating](http://blog.golang.org/share-memory-by-communicating)
