+++
date = "2018-03-26T12:39:53+09:00"
title = "Go におけるパッケージバージョニングについての提案"
draft = false
tags = ["tools", "versioning"]
+++

# Go におけるパッケージバージョニングについての提案
[A Proposal for Package Versioning in Go](https://blog.golang.org/versioning-proposal) by Russ Cox

## はじめに

8年前、Go チームは `goinstall` （のちの `go get`）や今日の Go 開発者にはお馴染みの分散化された URL のようなインポートパスを導入しました。`goinstall` をリリースした後、最初に尋ねられた質問の一つにバージョン情報をどのようにして盛り込むかというものがありました。私たちは分からなかったことを認めました。長いあいだ、私たちはパッケージのバージョニング問題はアドオンツールで解決するのが一番良いと信じており、人々にアドオンツールを作ることを勧めました。Go コミュニティは異なるアプローチでたくさんのツールを作りました。それらは私たちがその問題についてより理解することに役立ちましたが、2015年半ばまでにその問題には多くの解決方法があることが明らかとなりました。私たちは公式のツールとしてただ一つを採用する必要がありました。

2015年7月の GopherCon から同年秋まで行われたコミュニティでの議論の後、私たちはみな
Rust の Cargo におけるタグ付きのセマンティックバージョニングやマニフェスト、ロックファイル、どのバージョンを使うか決める [SAT ソルバー](https://research.swtch.com/version-sat) を例としたパッケージバージョニングのアプローチに従うという結論に至りました。このおおまかな計画に従い、`go` コマンドに統合するためのモデルとして役立てるつもりだった Dep 作りのために Sam Boyer はチームを先導しました。しかし、Cargo/Dep アプローチの意味するところについてより深く理解するにつれて、Go は一部の詳細（特に後方互換性）を変更するとこで恩恵を受けるであろうことが明らかとなりました。

## 互換性への影響

[Go 1](https://blog.golang.org/preview-of-go-version-1) の最も重要な新しい特徴は言語に関するものではありませんでした。Go 1 は後方互換性を重視していました。それまで約一か月毎に安定板のスナップショットをリリースしており、それぞれのリリースによる変更は互換性を極めて欠くものでした。Go 1 のリリース後すぐに、利益や採用において著しい加速を観測しました。[互換性に関する約束事](https://golang.org/doc/go1compat.html) により、開発者たちがプロダクションユースで Go を採用すると一層快適になると考えています。またその約束事が、今日 Go の人気が高まっている主な理由だと考えています。

一方で、[セマンティックバージョニング](http://semver.org/) は Go を含む多くの言語コミュニティにおいてソフトウェアのバージョンを記述するデファクトスタンダードとなりました。セマンティックバージョニングを用いることで、単一のメジャーバージョン内では後方のバージョンは前方のバージョンと後方互換性が保たれていることが期待されます。例えば、v1.2.3 は v1.2.1 や v1.1.5 と互換性が保たれていなければならないですが、v2.3.4 とは完全に互換性が保たれている必要はありません。

私たちが Go パッケージでセマンティックバージョニングを採用する場合、ほとんどの Go 開発者が期待する通り、インポートの互換性に関するルールは、異なるメジャーバージョンは異なるインポートパスを使っていなければならないということを要求します。この結果、v2.0.0 で始まるバージョンは `my/thing/v2/sub/pkg` のようなインポートパス内にメジャーバージョンを含む、といった _セマンティックインポートバージョニング_ が可能となりました。

一年前、インポートパス内にバージョン数を含むかどうかは主に趣味の問題だと私は強く考えており、それを含むことが特にエレガントだったかは懐疑的でした。しかしその決定は論理的には趣味の問題ではないと分かりました。つまり、インポートの互換性とセマンティックバージョニングは共にセマンティックインポートバージョニングを必要としているのです。私はこれに気づいたとき、論理的な必要性に驚きました。

[段階的なコードの修正](https://talks.golang.org/2016/refactor.article) や部分的なコードのアップグレードなどセマンティックインポートバージョニングとは独立した第2の論理的な考え方があることに気づき私はまた驚きました。巨大なプログラムおいて、特定の依存関係がある v1 から v2 に同時にアップデートすることを、プログラム内の全てのパッケージに期待するのは非現実的です。その代わりにプログラムの一部については v2 にアップグレードでき、その他の部分は v1 を使い続けることができるでしょう。しかしその一方で、ビルドされたバイナリは v1 と v2 の両方の依存関係を含んでいるでしょう。それらに同じインポートパスを与えると混乱につながり、私たちが _インポート一意性に関するルール（inport uniqueness rule）_ と呼んでいるもの、つまり異なるパッケージは異なるインポートパスを持っていなければならないというルールに違反します。部分的なコードのアップグレード、インポート一意性、_そして_ セマンティックバージョニングを行う唯一の方法はセマンティックインポートバージョニングなどを採用することです。

セマンティックインポートバージョニングなしでセマンティックバージョニングを用いるシステムをビルドすることはもちろん可能ですが、部分的なコードのアップグレードかインポート一意性のどちらかを諦めなければなりません。Cargo はインポート一意性を諦めることにより部分的なコードのアップグレードを可能としています。したがって、所定のインポートパスは大規模なビルドの異なる部分で異なる意味合いを持つことができます。Dep は部分的なコードのアップグレードを諦めることでインポート一意性を保証しています。したがって、大規模なビルドに関する全てのパッケージは所定の依存関係によって定められたただ一つのバージョンを探し出さなければならず、巨大なプログラムほどビルドできない可能性が高まります。Cargo は大規模なソフトウェア開発に不可欠な部分的なコードのアップグレードを主張のに適しています。Dep は同様にインポート一意性を主張するのに適しています。Go の現在のベンダリングサポートは使うのは煩雑で、インポート一意性に違反する可能性があります。違反した結果生じた問題は開発者とツールの両方にとってとても理解しにくいものでした。部分的なコードのアップグレードとインポート一意性のどちらかを選択する際は、選択しなかったことによる影響を予測する必要があります。セマンティックインポートバージョニングにより選択を避け、代わりに両方を維持することができます。

どのようなインポート互換性がバージョン選択を簡素化するかを発見したときも驚きました。これは、所定のビルドで用いるパッケージのバージョンを決める際の問題です。Cargo と Dep の制約により、バージョンを選択することと [ブール充足可能性を解決すること](https://research.swtch.com/version-sat) は同等に扱うことができ、それは有効なバージョン設定が存在するかどうかを判断するには高くつくかもしれないということ示唆しています。さらに、❝ベスト❞な設定を選択するための明確な基準なしでは、有効な設定がたくさんあるかもしれません。
代わりにインポート互換性に頼ることで Go は自明な線形時間のアルゴリズムを使用して、常に存在する唯一のベストな設定を探し出すことができます。
私が [_minimal version selection_](https://research.swtch.com/vgo-mvs) と呼んでいるこのアルゴリズムは、別々のロックファイルとマニフェストファイルの不必要とします。それらは
開発者とツールの両方が直接編集した単一の小さな設定ファイルで置き換えられ、再現可能なビルドは引き続きサポートされます。

Dep を使用した経験は互換性への影響を明らかにしています。Cargo とそれ以前のシステムに基づき、セマンティックバージョニングを採用する一環としてインポート互換性を諦めるために、私たちは Dep を設計しました。私はこれを故意に決めたとは考えていません。私たちはそれらとは別のシステムに従っただけです。Dep を直接使用した経験は、互換性のないインポートパスを許可することによってどのくらいの複雑さが生じるか正確に理解するのに役立ちました。セマンティックインポートバージョニングを導入することによってインポート互換性のルールを復活させることで、その複雑さを取り除き、よりシンプルなシステムにします。

## 進捗、プロトタイプ、提案

Dep was released in January 2017. Its basic model—code tagged with semantic versions, along with a configuration file that specified dependency requirements—was a clear step forward from most of the Go vendoring tools, and converging on Dep itself was also a clear step forward. I wholeheartedly encouraged its adoption, especially to help developers get used to thinking about Go package versions, both for their own code and their dependencies. While Dep was clearly moving us in the right direction, I had lingering concerns about the complexity devil in the details. I was particularly concerned about Dep lacking support for gradual code upgrades in large programs. Over the course of 2017, I talked to many people, including Sam Boyer and the rest of the package management working group, but none of us could see any clear way to reduce the complexity. (I did find many approaches that added to it.) Approaching the end of the year, it still seemed like SAT solvers and unsatisfiable builds might be the best we could do.

In mid-November, trying once again to work through how Dep could support gradual code upgrades, I realized that our old advice about import compatibility implied semantic import versioning. That seemed like a real breakthrough. I wrote a first draft of what became my [semantic import versioning](https://research.swtch.com/vgo-import) blog post, concluding it by suggesting that Dep adopt the convention. I sent the draft to the people I’d been talking to, and it elicited very strong responses: everyone loved it or hated it. I realized that I needed to work out more of the implications of semantic import versioning before circulating the idea further, and I set out to do that.

In mid-December, I discovered that import compatibility and semantic import versioning together allowed cutting version selection down to [minimal version selection](https://research.swtch.com/vgo-mvs). I wrote a basic implementation to be sure I understood it, I spent a while learning the theory behind why it was so simple, and I wrote a draft of the post describing it. Even so, I still wasn’t sure the approach would be practical in a real tool like Dep. It was clear that a prototype was needed.

In January, I started work on a simple `go` command wrapper that implemented semantic import versioning and minimal version selection. Trivial tests worked well. Approaching the end of the month, my simple wrapper could build Dep, a real program that made use of many versioned packages. The wrapper still had no command-line interface—the fact that it was building Dep was hard-coded in a few string constants—but the approach was clearly viable.

I spent the first three weeks of February turning the wrapper into a full versioned `go` command, `vgo`; writing drafts of a [blog post series introducing `vgo`](https://research.swtch.com/vgo); and discussing them with Sam Boyer, the package management working group, and the Go team. And then I spent the last week of February finally sharing `vgo` and the ideas behind it with the whole Go community.

In addition to the core ideas of import compatibility, semantic import versioning, and minimal version selection, the `vgo` prototype introduces a number of smaller
but significant changes motivated by eight years of experience with `goinstall` and `go get`: the new concept of a [Go module](https://research.swtch.com/vgo-module), which is a collection of packages versioned as a unit; [verifiable and verified builds](https://research.swtch.com/vgo-repro); and [version-awareness throughout the `go` command](https://research.swtch.com/vgo-cmd), enabling work outside `$GOPATH` and the elimination of (most) `vendor` directories.

The result of all of this is the [official Go proposal](https://golang.org/design/24301-versioned-go), which I filed last week.
Even though it might look like a complete implementation, it’s still just a prototype, one that we will all need to work together to complete. You can download and try the `vgo` prototype from [golang.org/x/vgo](https://golang.org/x/vgo), and you can read the [Tour of Versioned Go](https://research.swtch.com/vgo-tour) to get a sense of what using `vgo` is like.

## 今後の方針

The proposal I filed last week is exactly that: an initial proposal. I know there are problems with it that the Go team and I can’t see, because Go developers use Go in many clever ways that we don’t know about. The goal of the proposal feedback process is for us all to work together to identify and address the problems in the current proposal, to make sure that the final implementation that ships in a future Go release works well for as many developers as possible. Please point out problems on the [proposal discussion issue](https://golang.org/issue/24301). I will keep the [discussion summary](https://golang.org/issue/24301#issuecomment-371228742) and [FAQ](https://golang.org/issue/24301#issuecomment-371228664) updated as feedback arrives.

For this proposal to succeed, the Go ecosystem as a whole—and in particular today’s major Go projects—will need to adopt the import compatibility rule and semantic import versioning. To make sure that can happen smoothly, we will also be conducting user feedback sessions by video conference with projects that have questions about how to incorporate the new versioning proposal into their code bases or have feedback about their experiences. If you are interested in participating in such a session, please email Steve Francia at spf@golang.org.

We’re looking forward to (finally!) providing the Go community with a single, official answer to the question of how to incorporate package versioning into `go get`. Thanks to everyone who helped us get this far, and to everyone who will help us going forward. We hope that, with your help, we can ship something that Go developers will love.

By Russ Cox

## あわせて読みたい

* [The cover story](https://blog.golang.org/cover)
* [The App Engine SDK and workspaces (GOPATH)](https://blog.golang.org/the-app-engine-sdk-and-workspaces-gopath)
* [Organizing Go code](https://blog.golang.org/organizing-go-code)