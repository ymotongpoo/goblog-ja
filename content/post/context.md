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
