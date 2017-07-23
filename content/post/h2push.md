+++
date = "2011-03-24T21:12:42+09:00"
title = "HTTP/2 Server Push"
draft = false
tags = ["http", "technical"]
+++

# HTTP/2 Server Push
["HTTP/2 Server Push"](https://blog.golang.org/h2push) by Jaana Burcu Dogan, Tom Bergan

## はじめに

HTTP/2 is designed to address many of the failings of HTTP/1.x. Modern web pages use many resources: HTML, stylesheets, scripts, images, and so on. In HTTP/1.x, each of these resources must
be requested explicitly. This can be a slow process. The browser starts by fetching the HTML, then learns of more resources incrementally as it parses and evaluates the page. Since the server must wait for the browser to make each request, the network is often idle and underutilized.

To improve latency, HTTP/2 introduced _server_push_, which allows the server to push resources to the browser before they are explicitly requested. A server often knows many of the additional resources a page will need and can start pushing those resources as it responds to the initial request. This allows the server to fully utilize an otherwise idle network and improve page load times.

![h2push/serverpush](./h2push/serverpush.svg)

At the protocol level, HTTP/2 server push is driven by `PUSH_PROMISE` frames. A `PUSH_PROMISE` describes a request that the server predicts the browser will make in the near future. As soon as the browser receives a `PUSH_PROMISE`, it knows that the server will deliver the resource. If the browser later discovers that it needs this resource, it will wait for the push to complete rather than sending a new request. This reduces the time the browser spends waiting on the network.

## net/http における Server Push

Go 1.8 introduced support for pushing responses from an [`http.Server`](https://golang.org/pkg/net/http/#Server). This feature is available if the running server is an HTTP/2 server and the incoming connection uses HTTP/2. In any HTTP handler, you can assert if the http.ResponseWriter supports server push by checking if it implements the new [`http.Pusher`](https://golang.org/pkg/net/http/#Pusher) interface.

For example, if the server knows that `app.js` will be required to render the page, the handler can initiate a push if `http.Pusher` is available:

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

The Push call creates a synthetic request for `/app.js`, synthesizes that request into a `PUSH_PROMISE` frame, then forwards the synthetic request to the server's request handler, which will generate the pushed response. The second argument to Push specifies additional headers to include in the `PUSH_PROMISE`. For example, if the response to `/app.js` varies on Accept-Encoding, then the `PUSH_PROMISE` should include an Accept-Encoding value:

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

A fully working example is available at:

```
$ go get golang.org/x/blog/content/h2push/server
```

If you run the server and load [https://localhost:8080](https://localhost:8080),
your browser's developer tools should show that `app.js` and `style.css` were pushed by the server.

![h2push/networktimeline](./h2push/networktimeline.png)

## 応答する前にPushし始める

It's a good idea to call the Push method before sending any bytes of the response. Otherwise it is possible to accidentally generate duplicate responses. For example, suppose you write part of an HTML response:

```
<html>
<head>
    <link rel="stylesheet" href="a.css">...
```

Then you call Push("a.css", nil). The browser may parse this fragment of HTML before it receives your PUSH_PROMISE, in which case the browser will send a request for `a.css` in addition to receiving your `PUSH_PROMISE`. Now the server will generate two responses for `a.css`. Calling Push before writing the response avoids this possibility entirely.

## Server Push の使い時

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

With Go 1.8, the standard library provides out-of-the-box support for HTTP/2 Server Push, giving you more flexibility to optimize your web applications.

Go to our [HTTP/2 Server Push demo](https://http2.golang.org/serverpush) page to see it in action.

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