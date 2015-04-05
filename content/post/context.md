+++
date = "2014-07-29T09:00:00+09:00"
draft = false
title = "Goの並行パターン：コンテキスト (Go Concurrency Pattern: Context)"
+++

# Goの並行パターン：コンテキスト
[Go Concurrency Pattern: Context](https://blog.golang.org/context) by Sameer Ajmani

## はじめに
Goで書かれたサーバでは、サーバに来たリクエストはそれぞれそれ自身のゴルーチンで処理されます。
