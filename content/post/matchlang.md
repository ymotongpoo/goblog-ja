+++
date = "2016-02-09T21:55:32+09:00"
draft = true
title = "matchlang"

+++

# Goにおける言語とロケールのマッチング
[Language and Locale Matching in Go](https://blog.golang.org/matchlang) By Marcel van Lohuizen

## はじめに
ウェブサイトのような、ユーザーインターフェースで複数の言語をサポートするアプリケーションを考えてみましょう。
ユーザーが望む言語のリストがある場合、アプリケーションはどの言語を表示すべきか決めなければなりません。
このとき、アプリケーションがサポートする言語とユーザーが好む言語の間で最適な組み合わせを選ばなければなりません。
この記事ではなぜこの決定が難しいか、そしてどうやってGoがそれを手助けできるかを説明します。

## 言語タグ
言語タグ、あるいはロケール識別子、は使用されている言語と方言をコンピュータが理解できる形でにした識別子です。
もっともよく知られたものとしては IETF BCP 47 という標準で、Goのライブラリもこの標準に準じています。
BCP 47 で定義されている言語タグと、それらが表す言語や方言をいくつか例を挙げてみましょう。

| タグ   | 説明        |
|:------|:------------|
|en     |英語         |
|en-US  |アメリカ英語   |
|cmn    |標準中国語    |
|zh     |中国語（通常は標準語）|
|nl     |オランダ語    |
|nl-BE  |フラマン語    |
|es-419 |ラテンアメリカスペイン語|
|az, az-Latn |ともにラテン文字で書かれたアゼルバイジャン語|
|az-Arab |アラビア文字で書かれたアゼルバイジャン語|

言語タグは一般的に言語コード（上記での“en”, “cmn”, “zh”, “nl”, “az”）が来た後に付加的な文字に関する副タグ（“-Arab”）や
地域に関する副タグ（“-US”, “-BE”, “-419”）、変数の副タグ（オックスフォード英語大辞典でのスペルのための “-oxendict”）、あるいは
拡張副タグ（電話帳順のための “-u-co-phonebk”）が続きます。
もっとも一般的な形式は副タグが省略された形、たとえば “az-Latn-AZ” であれば “az” です。

言語タグがもっとも使われる場所は、システムがサポートしている言語の一覧からユーザーが好みの言語を選択するときでしょう。
たとえば、（アフリカーンス語が選択できない場合に）アフリカーンス語を望んでいるユーザーに対しシステムはオランダ語を表示するという決定をする場合です。
このような対応は言語間の包含性に関するデータを参照するというプロセスが関わってきます。
この対応から得られたタグは、その後言語特有のリソース、例えば翻訳、並び順、タイトルなどの大文字小文字を自動で変えるアルゴリズムなどを
取得するために使われます。これらの処理はまた別の対応を必要とします。たとえば、ポルトガル語には決まった並び順がないため、
並び順を処理するパッケージはデフォルトのもの、すなわち「ルート」の言語の並び順にフォールバックすることになるでしょう。

## 言語の対応に関する厄介な性質
言語タグを扱うには注意が必要です。その理由は、自然言語の境界があいまいであること、言語タグの標準の発展の歴史によるもの、などが挙げられます。
この節では言語タグを扱う際の厄介な側面をいくつかご紹介します。


### 異なる言語コードのタグが同じ言語を表している

歴史的かつ政治的な理由で、多くの言語コードは時とともに変遷していて、古い言語コードと新しい言語コードが共存していました。
しかし、古いものも新しいものも同じ言語を参照しています。たとえば、標準中国語を表す公式な言語コードは “cmn” ですが、 “zh” がずっと
広く使われています。 “zh” は公式にはいわゆるマクロ言語として中国語全般を表すために予約されています。
マクロ言語用の言語タグは、しばしばその言語群の中でもっとも良く話されている言語のコードとして使われます。

### 言語コードの対応だけでは不十分

たとえばアゼルバイジャン語（“az”）は国によって異なる文字で書かれています。"az-Latn" はラテン文字、"az-Arab" はアラビア文字、
“az-Cyrl” はキリル文字です。もし “az-Arab” を単純に “az” に置き換えてしまうと、結果としてラテン文字が表示されて、
アラビア文字で書かれたアゼルバイジャン語しか理解できない人には意味のないものになってしまうでしょう。

異なる地域であることも異なる文字が使われる可能性を示唆します。たとえば “zh-TW” と “zh-SG” はそれぞれ繁体字中国語、
簡体字中国語であることを示しています。他の例としては “sr” （セルビア語）はデフォルトではキリル文字ですが、
“sr-RU” （ロシアで書かれるセルビア語）はラテン文字なのです！似た例はキルギスや他の言語でも見られます。

副タグを無視すると、ユーザーにとって意味の分からないものが表示されるかもしれません。

### ユーザーが選択していない言語が最適な場合もある

もっとも普及したノルウェー語（“nb”）は、デンマーク語と見分けが付きません。もしノルウェー語が選択できないのであれば、
デンマーク語が次点として選ばれるべきでしょう。同様に、スイスのドイツ語（“gsw”）を選択したユーザーに
ドイツ語（“de”）が表示されても問題無いでしょう。しかし、その逆はまったく当てはまりません。
ウイグル語を選択したユーザーに対しては英語よりも中国語にフォールバックさせたほうが良いでしょう。
ユーザーが選択した言語がサポートされていない場合に、必ずしも英語にフォールバックすることが最適ではないのです。

### 翻訳よりも言語の選択の方が重要

あるユーザーがデンマーク語を第1の選択肢に、ドイツ語を第2の選択肢にしたとしましょう。
もしアプリケーションがドイツ語を選択するならば、コンテンツをドイツ語に翻訳するだけではなく、照合処理も（デンマーク語ではなく）ドイツ語の
ルールを使わなければいけません。そうしないと、たとえば動物のリストを並べるときに “Bär” （日：クマ）が “Äffin” （日：メスザル）よりも
前に来てしまいます。

ユーザーが選択した言語の中からサポートする言語を選択するというのは、ハンドシェイクアルゴリズムに似ています。
まずあなたがやり取りするためのプロトコル（言語）を決定し、そのあとはセッションが続く間はすべてのコミュニケーションを
そのプロトコルで行うことに終始します。

### フォールバックに「親」の言語を使うのは簡単なことではない

あなたのアプリケーションがアンゴラのポルトガル語（“pt-AO”）をサポートしているとしましょう。
[golang.org/x/text](http://golang.org/x/text) 内のパッケージでは、この方言に対しての、照合や表示といった、特別なサポートはありません。
そのような状況で採るべき正しい行動は、もっとも近い親方言を対応させることです。言語は階層的な関係にあり、特定の言語には、
より一般的な親の方言があります。たとえば、“en-GB-oxendict” の親は “en-GB” であり、その親は “en” で、さらにその親は未定義の言語を表す
“und” となります。これはルート言語としても知られています。照合においては、ポルトガル語には特定の並び順がないため、照合用パッケージは
ルート言語の並び順を選択するでしょう。表示用パッケージでアンゴラのポルトガル語に最も近い親は、ヨーロッパポルトガル語（“pt-PT”）で、
よりわかりやす “pt” ではありません。こちらはブラジルのポルトガル語を表します。

一般的に、親子関係は簡単ではありません。もう少し例を挙げましょう。 “es-CL” の親は “es-419” で、 “zh-TW” の親は “zh-Hant”、その親は “und” です。
祖語を選択しようと単純に副タグを取っただけの処理をすると、ユーザーが理解できない「方言」を選択してしまうかもしれません。

## Goでの言語のマッチング

Goのパッケージ [golang.org/x/text/language](http://golang.org/x/text/language) は言語タグに関するBCP 47の標準を実装し、
共通ロケールデータレポジトリ（CLDR）にあるデータに基づいて、どの言語を使うべきかを判断するサポートを追加しています。

ユーザーの選択した言語とアプリケーションがサポートしている言語との対応をするサンプルプログラムを示します。

```
package main

import (
    "fmt"

    "golang.org/x/text/language"
    "golang.org/x/text/language/display"
)

var userPrefs = []language.Tag{
    language.Make("gsw"), // Swiss German
    language.Make("fr"),  // French
}

var serverLangs = []language.Tag{
    language.AmericanEnglish, // en-US fallback
    language.German,          // de
}

var matcher = language.NewMatcher(serverLangs)

func main() {
    tag, index, confidence := matcher.Match(userPrefs...)

    fmt.Printf("best match: %s (%s) index=%d confidence=%v\n",
        display.English.Tags().Name(tag),
        display.Self.Name(tag),
        index, confidence)
    // best match: German (Deutsch) index=1 confidence=High
}
```

## 言語タグを作成する

The simplest way to create a language.Tag from a user-given language code string is with language.Make. It extracts meaningful information even from malformed input. For example, “en-USD” will result in “en” even though USD is not a valid subtag.

Make doesn’t return an error. It is common practice to use the default language if an error occurs anyway so this makes it more convenient. Use Parse to handle any error manually.

The HTTP Accept-Language header is often used to pass a user’s desired languages. The ParseAcceptLanguage function parses it into a slice of language tags, ordered by preference.

By default, the language package does not canonicalize tags. For example, it does not follow the BCP 47 recommendation of eliminating scripts if it is the common choice in the “overwhelming majority”. It similarly ignores CLDR recommendations: “cmn” is not replaced by “zh” and “zh-Hant-HK” is not simplified to “zh-HK”. Canonicalizing tags may throw away useful information about user intent. Canonicalization is handled in the Matcher instead. A full array of canonicalization options are available if the programmer still desires to do so.

## Matching User-Preferred Languages to Supported Languages

A Matcher matches user-preferred languages to supported languages. Users are strongly advised to use it if they don’t want to deal with all the intricacies of matching languages.

The Match method may pass through user settings (from BCP 47 extensions) from the preferred tags to the selected supported tag. It is therefore important that the tag returned by Match is used to obtain language-specific resources. For example, “de-u-co-phonebk” requests phone-book ordering for German. The extension is ignored for matching, but is used by the collate package to select the respective sorting order variant.

A Matcher is initialized with the languages supported by an application, which are usually the languages for which there are translations. This set is typically fixed, allowing a matcher to be created at startup. Matcher is optimized to improve the performance of Match at the expense of initialization cost.

The language package provides a predefined set of the most commonly used language tags that can be used for defining the supported set. Users generally don’t have to worry about the exact tags to pick for supported languages. For example, AmericanEnglish (“en-US”) may be used interchangeably with the more common English (“en”), which defaults to American. It is all the same for the Matcher. An application may even add both, allowing for more specific American slang for “en-US”.

## Matching Example

Consider the following Matcher and lists of supported languages:

```
var supported = []language.Tag{
    language.AmericanEnglish,    // en-US: first language is fallback
    language.German,             // de
    language.Dutch,              // nl
    language.Portuguese          // pt (defaults to Brazilian)
    language.EuropeanPortuguese, // pt-pT
    language.Romanian            // ro
    language.Serbian,            // sr (defaults to Cyrillic script)
    language.SerbianLatin,       // sr-Latn
    language.SimplifiedChinese,  // zh-Hans
    language.TraditionalChinese, // zh-Hant
}
var matcher = language.NewMatcher(supported)
```

Let's look at the matches against this list of supported languages for various user preferences.

For a user preference of "he" (Hebrew), the best match is "en-US" (American English). There is no good match, so the matcher uses the fallback language (the first in the supported list).

For a user preference of "hr" (Croatian), the best match is "sr-Latn" (Serbian with Latin script), because, once they are written in the same script, Serbian and Croatian are mutually intelligible.

For a user preference of "ru, mo" (Russian, then Moldavian), the best match is "ro" (Romanian), because Moldavian is now canonically classified as "ro-MD" (Romanian in Moldova).

For a user preference of "zh-TW" (Mandarin in Taiwan), the best match is "zh-Hant" (Mandarin written in Traditional Chinese), not "zh-Hans" (Mandarin written in Simplified Chinese).

For a user preference of "af, ar" (Afrikaans, then Arabic), the best match is "nl" (Dutch). Neither preference is supported directly, but Dutch is a significantly closer match to Afrikaans than the fallback language English is to either.

For a user preference of "pt-AO, id" (Angolan Portuguese, then Indonesian), the best match is "pt-PT" (European Portuguese), not "pt" (Brazilian Portuguese).

For a user preference of "gsw-u-co-phonebk" (Swiss German with phone-book collation order), the best match is "de-u-co-phonebk" (German with phone-book collation order). German is the best match for Swiss German in the server's language list, and the option for phone-book collation order has been carried over.

## Confidence Scores

Go uses coarse-grained confidence scoring with rule-based elimination. A match is classified as Exact, High (not exact, but no known ambiguity), Low (probably the correct match, but maybe not), or No. In case of multiple matches, there is a set of tie-breaking rules that are executed in order. The first match is returned in the case of multiple equal matches. These confidence scores may be useful, for example, to reject relatively weak matches. They are also used to score, for example, the most likely region or script from a language tag.

Implementations in other languages often use more fine-grained, variable-scale scoring. We found that using coarse-grained scoring in the Go implementation ended up simpler to implement, more maintainable, and faster, meaning that we could handle more rules.

## Displaying Supported Languages

The golang.org/x/text/language/display package allows naming language tags in many languages. It also contains a “Self” namer for displaying a tag in its own language.

For example:

```
    var supported = []language.Tag{
        language.English,            // en
        language.French,             // fr
        language.Dutch,              // nl
        language.Make("nl-BE"),      // nl-BE
        language.SimplifiedChinese,  // zh-Hans
        language.TraditionalChinese, // zh-Hant
        language.Russian,            // ru
    }

    en := display.English.Tags()
    for _, t := range supported {
        fmt.Printf("%-20s (%s)\n", en.Name(t), display.Self.Name(t))
    }
```

prints

```
English              (English)
French               (français)
Dutch                (Nederlands)
Flemish              (Vlaams)
Simplified Chinese   (简体中文)
Traditional Chinese  (繁體中文)
Russian              (русский)
```

In the second column, note the differences in capitalization, reflecting the rules of the respective language.

## 結論

At first glance, language tags look like nicely structured data, but because they describe human languages, the structure of relationships between language tags is actually quite complex. It is often tempting, especially for English-speaking programmers, to write ad-hoc language matching using nothing other than string manipulation of the language tags. As described above, this can produce awful results.

Go's golang.org/x/text/language package solves this complex problem while still presenting a simple, easy-to-use API. Enjoy.

By Marcel van Lohuizen