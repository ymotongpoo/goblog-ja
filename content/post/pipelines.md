+++
date = "2014-03-13T08:19:15+09:00"
draft = true
title = "Goの並行パターン：パイプラインとキャンセル (Go Concurrency Patterns: Pipelines and cancellation)"
tags = ["concurrency", "pipeline", "cancellation"]
+++

# Goの並行パターン：パイプラインとキャンセル
[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines) by Sameer Ajmani

## はじめに
Goの並行性に関する基本要素によって、I/Oや複数のCPIを効率的に使うことができるストリーミングデータパイプラインを
簡単に構築することができます。この記事ではそのようなパイプラインの例を紹介し、操作が失敗したときに発生する
繊細な事柄にハイライトを当て、また失敗に綺麗に対応するテクニックを紹介します。
