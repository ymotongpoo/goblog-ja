+++
date = "2011-03-24T21:12:42+09:00"
title = "HTTP/2 サーバプッシュ（HTTP/2 Server Push）"
draft = false
tags = ["http", "technical"]
+++

# HTTP/2 サーバプッシュ
["HTTP/2 Server Push"](https://blog.golang.org/h2push) by Jaana Burcu Dogan, Tom Bergan

## はじめに

HTTP/2 は HTTP/1.x における多くの欠点を扱うために設計されています。現代の Web ページは多くのリソースを使用します: HTML、スタイルシート、スクリプト、画像、など。HTTP/1.x では、そのリソースそれぞれが明示的にリクエストされなければなりませんでした。これでは処理が遅くなります。ブラウザが HTML を取得し始めると、ページをパースして評価するときに初めて追加のリソースが必要であると認識します。サーバはブラウザがそれぞれのリクエストを生成するのを待たないといけないので、ネットワークは頻繁にアイドル状態となり活動を休止します。

遅延を改善するために HTTP/2 は、ブラウザが明示的にリソースをリクエストする前にサーバがブラウザに対しリソースをプッシュできる*サーバプッシュ*を導入しました、サーバはページが沢山の追加リソースを頻繁に必要とすることを分かっているため、最初の要求に対するレスポンスとしてそれらのリソースをプッシュし始めることができます。これでサーバがアイドル状態のネットワークを十分に活用でき、ページのロード時間を改善できます。

![h2push/serverpush](./h2push/serverpush.svg)

プロトコルのレベルでは、HTTP/2 サーバプッシュは `PUSH_PROMISE` フレームドリブンです。`PUSH_PROMISE` はブラウザが近い将来リクエストするとサーバが予測するリクエストを記述します。ブラウザが`PUSH_PROMISE` を受け取るとすぐに、サーバがそのリソースを届けるつもりであるとブラウザは理解します。ブラウザがこのリソースが必要であると遅れて気づいた場合は、ブラウザは新しいリクエストを送らずにプッシュが完了するのを待つでしょう。

## net/http におけるサーバプッシュ

Go1.8は [`http.Server`](https://golang.org/pkg/net/http/#Server) からのレスポンスをプッシュするためのサポートを導入しました。この特徴は動作中のサーバが HTTP/2 サーバであり HTTP/2 を使用するコネクションが来た場合に利用されます。任意の HTTP ハンドラの中で、http.ResponseWriter が新しい [`http.Pusher`](https://golang.org/pkg/net/http/#Pusher) インターフェースを実装しているか確認することによりサーバプッシュをサポートしているかどうか、あなたは強く主張することができます。

例えば、ページを描画するのに `app.js` が要求されるだろうとサーバが分かっている場合、ハンドラは `http.Pusher` が利用であればプッシュを開始することができます:

```
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    if pusher, ok := w.(http.Pusher); ok {
        // Push is supported.
        if err := pusher.Push("/app.js", nil); err != nil {
            log.Printf("Failed to push: %v", err)
        }
    }
    // ...
})
```

Push 呼び出しは `/app.js` のための統合リクエストを作成し、`PUSH_PROMISE` フレームの中にそのリクエストを統合し、統合リクエストをプッシュされるレスポンスを生成するサーバのリクエストハンドラに転送します。Push の2つ目の引数は `PUSH_PROMISE` に含める追加のヘッダを明示します。例えば、`/app.js` へのレスポンスが Accept-Encoding よって異なる場合、`PUSH_PROMISE` は Accept-Encoding 値を含むべきです:

```
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    if pusher, ok := w.(http.Pusher); ok {
        // Push is supported.
        options := &http.PushOptions{
            Header: http.Header{
                "Accept-Encoding": r.Header["Accept-Encoding"],
            },
        }
        if err := pusher.Push("/app.js", options); err != nil {
            log.Printf("Failed to push: %v", err)
        }
    }
    // ...
})
```

完全な動作例は以下のコマンドで利用可能です:

```
$ go get golang.org/x/blog/content/h2push/server
```

サーバを動作させ [https://localhost:8080](https://localhost:8080) をロードすると、ブラウザの開発者ツールには `app.js` と `style.css` がサーバからプッシュされたと表示されるはずです。

![h2push/networktimeline](./h2push/networktimeline.png)

## レスポンスの前にプッシュし始める

It's a good idea to call the Push method before sending any bytes of the response. Otherwise it is possible to accidentally generate duplicate responses. For example, suppose you write part of an HTML response:

```
<html>
<head>
    <link rel="stylesheet" href="a.css">...
```

Then you call Push("a.css", nil). The browser may parse this fragment of HTML before it receives your PUSH_PROMISE, in which case the browser will send a request for `a.css` in addition to receiving your `PUSH_PROMISE`. Now the server will generate two responses for `a.css`. Calling Push before writing the response avoids this possibility entirely.

## サーバプッシュの使い時

Consider using server push any time your network link is idle. Just finished sending the HTML for your web app? Don't waste time waiting, start pushing the resources your client will need. Are you inlining resources into your HTML file to reduce latency? Instead of inlining, try pushing. Redirects are another good time to use push because there is almost always a wasted round trip while the client follows the redirect. There are many possible scenarios for using push -- we are only getting started.

We would be remiss if we did not mention a few caveats. First, you can only push resources your server is authoritative for -- this means you cannot push resources that are hosted on third-party servers or CDNs. Second, don't push resources unless you are confident they are actually needed by the client, otherwise your push wastes bandwidth. A corollary is to avoid pushing resources when it's likely that the client already has those resources cached. Third, the naive approach of pushing all resources on your page often makes performance worse. When in doubt, measure.

The following links make for good supplemental reading:

* [HTTP/2 Push: The Details](https://calendar.perfplanet.com/2016/http2-push-the-details/)
* [Innovating with HTTP/2 Server Push](https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/)
* [Cache-Aware Server Push in H2O](https://github.com/h2o/h2o/issues/421)
* [The PRPL Pattern](https://developers.google.com/web/fundamentals/performance/prpl-pattern/)
* [Rules of Thumb for HTTP/2 Push](https://docs.google.com/document/d/1K0NykTXBbbbTlv60t5MyJvXjqKGsCVNYHyLEXIxYMv0)
* [Server Push in the HTTP/2 spec](https://tools.ietf.org/html/rfc7540#section-8.2)

## 結論

Go1.8では、標準ライブラリは HTTP/2 のための独創的なサポートを提供しており、あなたの Web アプリケーションを最適化できるようさらなる柔軟性を与えます。

[HTTP/2 サーバプッシュデモ](https://http2.golang.org/serverpush) ページではそれが実際に動作する様子を見ることができます。

## あわせて読みたい
* [HTTP/2 Push: The Details](https://calendar.perfplanet.com/2016/http2-push-the-details/)
* [Innovating with HTTP/2 Server Push](https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/)
* [Cache-Aware Server Push in H2O](https://github.com/h2o/h2o/issues/421)
* [The PRPL Pattern](https://developers.google.com/web/fundamentals/performance/prpl-pattern/)
* [Rules of Thumb for HTTP/2 Push](https://docs.google.com/document/d/1K0NykTXBbbbTlv60t5MyJvXjqKGsCVNYHyLEXIxYMv0)
* [Server Push in the HTTP/2 spec](https://tools.ietf.org/html/rfc7540#section-8.2)
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
* [C? Go? Cgo!](https://blog.golang.org/c-go-cgo)
* [JSON and Go](https://blog.golang.org/json-and-go)
* [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
* [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)
* [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
* [JSON-RPC: a tale of interfaces](https://blog.golang.org/json-rpc-tale-of-interfaces)