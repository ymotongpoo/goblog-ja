+++
date = "2013-11-26T10:49:22+09:00"
draft = true
title = "Goでの文字列の正規化 (Text normalization in Go)"
tags = ["strings", "bytes", "runes", "characters"]
+++

# Goでの文字列の正規化

[Text normalization in Go](https://blog.golang.org/normalization) by By Marcel van Lohuizen

## はじめに

先の[記事](./strings/)では、Goでの文字列、バイト、文字について説明していました。
私は `go.text` レポジトリ（訳注：現在は `golang.org/x/text` パッケージ群、以下原文で `go.text` の部分は置き換える。）で多言語文字列処理向けの様々なパッケージの開発に関わってきました。
これらのパッケージのいくつかは別のブログポストに譲って、この記事では [go.text/unicode/norm](http://godoc.org/code.google.com/p/go.text/unicode/norm) （訳注：現在は [golang.org/x/text/unicode/norm](http://godoc.org/golang.org/x/text/unicode/norm)）に焦点を当てたいと思います。
このパッケージは、先の[文字列に関する記事](./strings/)、そして本記事のタイトルとなっている、文字列の正規化を扱います。
正規化は生のバイト列よりも高水準での抽象化を扱います。

正規化についてのすべてを知りたければ、[Unicode標準の付録15](http://unicode.org/reports/tr15/)を読むのが良いでしょう。
より読みやすい記事としては、対応する[Wikipediaのページ](http://en.wikipedia.org/wiki/Unicode_equivalence)（訳注：[日本語版](https://ja.wikipedia.org/wiki/Unicode%E6%AD%A3%E8%A6%8F%E5%8C%96)）があります。
ここでは、正規化がどのようにGoに関わっているかに焦点を当てます。

## 正規化とは何か

同じ文字列を表現するときにいくつかの方法があることがしばしばあります。たとえば、é（eのアキュート）は文字列内で単一のルーン（"\u00e9"）または
'e'のあとにアキュート・アクセントが続いたもの（"e\u0301"）として表現されます。Unicode標準によれば、この2つの表現はどちらも「正準等価」
であり、同等に扱われるべきです。

これら2つの表現の等価性を調べるために1バイトごとに比較していたのでは正しい結果は導けません。Unicodeでは2つの表現が正準に等価で、
同じ正規形に正規化される場合に、そのバイト表現が同じになるような正規形のセットを定義しています。

またUnicodeでは同じ文字を表すけれども見た目が異なる表現を同等とみなす「互換等価」を定義しています。
たとえば上付き文字の数字 '⁹' と通常の数字 '9' はこの形式では等価です。

これら2つ等価な形式に対して、Unicodeでは合成と分解を定義しています。
合成は、結合して1つのルーンにできる複数のルーンをその1つのルーンにい置きv換えることです。
分解は、ルーンを要素に切り離すことを指します。すべてNFから始まる次の表は、Unicodeコンソーシアムが各形式を識別する際に使っているものです。v

|            |**合成**   |**分解**     |
|:-----------|:---------|:-----------|
|**正準等価** |NFC       |NFD         |
|**互換等価** |NFKC      |NFKD        |

## Goの正規化に対するアプローチ

文字列に関する記事で言及したように、Goでは文字列内の文字が正規化されていることを保証していません。
しかしながら、`golang.org/x/text` パッケージがそれを補填してくれます。
たとえば [collate](http://godoc.org/golang.org/x/text/collate) パッケージは、Go言語特有の方法で文字列を順場に並べるパッケージで、
これは正規化されていない文字列でも正しく動作します。
`golang.org/x/text` 内のパッケージは必ずしも入力が正規化されている必要はありませんが、一般的に一貫した結果を得るためには
正規化が必要でしょう。

正規化のコストはタダではないですが速いです。特に、照合や検索の場合、または文字列がNFDかNFCのいずれかで、バイトを並び替えなくても分解するだけで
NFDになる場合は顕著です。実際に、99.98%のウェブ上のHTMLページのコンテンツはNFC形式です。（マークアップは含めていません。
含めた場合はその割合はより大きくなります。）ほぼ間違いなくたいていのNFCは（メモリの確保を必要とする）並び替えの必要なく分解するだけでNFDになります。
また、並び替えが必要な場合の検出も効率的なので、それが必要なまれな区画に対してだけ並び替えを行うことで時間を節約することが出来ます。

より効率よくするために、照合（collate）のパッケージは通常は `norm` パッケージを直接は使わず、代わりに自身が持っている表の中に
正規化に関する情報をインタリーブするために `norm` パッケージを使います。
並び替えと正規化をインタリーブすることで、パフォーマンスに影響をあたえることなくその場で実行することが可能になります。
オンザフライでの正規化のコストは事前に事前に文字列を正規化する必要がないことで補償され、編集時には正規化形式になっていることを保証します。
特に後者は厄介です。たとえば、2つのNFCで正規化された文字を合成してもNFCになるとは限らないからです。

もちろん、よくあることですが、事前に文字列が正規化済みであることがわかっているのであれば、明白なオーバーヘッドは避けるべきでもあります。

## 何が困るのか

これまで正規化をできれば避けたいという話をしてきましたが、そもそもなぜそれが懸念事項になるのか疑問の人もいるでしょう。
その理由は、正規化が必要で、正規化が何か、そして正規化を正しくする方法を理解することが重要である場合があるからです。

これらについて議論する前に、まず「文字」という概念を明確にしなければなりません。

## 文字とは何か

文字列に関する記事で説明したとおり、文字は複数のルーンに渡ることがあります。たとえば 'e' と '◌́'（アキュート "\u0301"）は合成して
'é' という形式（NFDでは "e\u0301" ）になります。この2つのルーンをまとめて1つの文字を表しています。
文字の定義は実装によって変わります。正規化においては文字を、他のルーンを変更したり後ろ向きに合成しないルーンと定義した開始ルーンから始まり、
（通常はアクセントなどの）修飾などを行う後続ルーンによるルーン列として定義しています。後続ルーンの列は空になりえます。
正規化アルゴリズムは一度に1文字を処理します。

Theoretically, there is no bound to the number of runes that can make up a Unicode character. In fact, there are no restrictions on the number of modifiers that can follow a character and a modifier may be repeated, or stacked. Ever seen an 'e' with three acutes? Here you go: 'é́́'. That is a perfectly valid 4-rune character according to the standard.

As a consequence, even at the lowest level, text needs to be processed in increments of unbounded chunk sizes. This is especially awkward with a streaming approach to text processing, as used by Go's standard Reader and Writer interfaces, as that model potentially requires any intermediate buffers to have unbounded size as well. Also, a straightforward implementation of normalization will have a O(n²) running time.

There are really no meaningful interpretations for such large sequences of modifiers for practical applications. Unicode defines a Stream-Safe Text format, which allows capping the number of modifiers (non-starters) to at most 30, more than enough for any practical purpose. Subsequent modifiers will be placed after a freshly inserted Combining Grapheme Joiner (CGJ or U+034F). Go adopts this approach for all normalization algorithms. This decision gives up a little conformance but gains a little safety.

## Writing in normal form

Even if you don't need to normalize text within your Go code, you might still want to do so when communicating to the outside world. For example, normalizing to NFC might compact your text, making it cheaper to send down a wire. For some languages, like Korean, the savings can be substantial. Also, some external APIs might expect text in a certain normal form. Or you might just want to fit in and output your text as NFC like the rest of the world.

To write your text as NFC, use the [unicode/norm](http://godoc.org/code.google.com/p/go.text/unicode/norm) package to wrap your io.Writer of choice:

```
wc := norm.NFC.Writer(w)
defer wc.Close()
// write as before...
```

If you have a small string and want to do a quick conversion, you can use this simpler form:

```
norm.NFC.Bytes(b)
```

Package norm provides various other methods for normalizing text. Pick the one that suits your needs best.

## Catching look-alikes

Can you tell the difference between 'K' ("\u004B") and 'K' (Kelvin sign "\u212A") or 'Ω' ("\u03a9") and 'Ω' (Ohm sign "\u2126")? It is easy to overlook the sometimes minute differences between variants of the same underlying character. It is generally a good idea to disallow such variants in identifiers or anything where deceiving users with such look-alikes can pose a security hazard.

The compatibility normal forms, NFKC and NFKD, will map many visually nearly identical forms to a single value. Note that it will not do so when two symbols look alike, but are really from two different alphabets. For example the Latin 'o', Greek 'ο', and Cyrillic 'о' are still different characters as defined by these forms.

## Correct text modifications

The norm package might also come to the rescue when one needs to modify text. Consider a case where you want to search and replace the word "cafe" with its plural form "cafes".  A code snippet could look like this.

```
s := "We went to eat at multiple cafe"
cafe := "cafe"
if p := strings.Index(s, cafe); p != -1 {
    p += len(cafe)
    s = s[:p] + "s" + s[p:]
}
fmt.Println(s)
```

This prints "We went to eat at multiple cafes" as desired and expected. Now consider our text contains the French spelling "café" in NFD form:

```
s := "We went to eat at multiple cafe\u0301"
```

Using the same code from above, the plural "s" would still be inserted after the 'e', but before the acute, resulting in  "We went to eat at multiple cafeś".  This behavior is undesirable.

The problem is that the code does not respect the boundaries between multi-rune characters and inserts a rune in the middle of a character.  Using the norm package, we can rewrite this piece of code as follows:

```
s := "We went to eat at multiple cafe\u0301"
cafe := "cafe"
if p := strings.Index(s, cafe); p != -1 {
    p += len(cafe)
    if bp := norm.FirstBoundary(s[p:]); bp > 0 {
        p += bp
    }
    s = s[:p] + "s" + s[p:]
}
fmt.Println(s)
```

This may be a contrived example, but the gist should be clear. Be mindful of the fact that characters can span multiple runes. Generally these kinds of problems can be avoided by using search functionality that respects character boundaries (such as the planned go.text/search package.)

## Iteration

Another tool provided by the norm package that may help dealing with character boundaries is its iterator, [norm.Iter](http://godoc.org/code.google.com/p/go.text/unicode/norm#Iter). It iterates over characters one at a time in the normal form of choice.

## Performing magic

As mentioned earlier, most text is in NFC form, where base characters and modifiers are combined into a single rune whenever possible.  For the purpose of analyzing characters, it is often easier to handle runes after decomposition into their smallest components. This is where the NFD form comes in handy. For example, the following piece of code creates a transform.Transformer that decomposes text into its smallest parts, removes all accents, and then recomposes the text into NFC:

```
import (
    "unicode"

    "golang.org/x/text/transform"
    "golang.org/x/text/unicode/norm"
)

isMn := func(r rune) bool {
    return unicode.Is(unicode.Mn, r) // Mn: nonspacing marks
}
t := transform.Chain(norm.NFD, transform.RemoveFunc(isMn), norm.NFC)
```

The resulting Transformer can be used to remove accents from an io.Reader of choice as follows:

```
r = transform.NewReader(r, t)
// read as before ...
```

This will, for example, convert any mention of "cafés" in the text to "cafes", regardless of the normal form in which the original text was encoded.

## Normalization info

As mentioned earlier, some packages precompute normalizations into their tables to minimize the need for normalization at run time. The type norm.Properties provides access to the per-rune information needed by these packages, most notably the Canonical Combining Class and decomposition information. Read the [documentation](http://godoc.org/code.google.com/p/go.text/unicode/norm/#Properties) for this type if you want to dig deeper.

## Performance

To give an idea of the performance of normalization, we compare it against the performance of strings.ToLower. The sample in the first row is both lowercase and NFC and can in every case be returned as is. The second sample is neither and requires writing a new version.

|*Input              |*ToLower|*NFC Append|*NFC Transform|*NFC Iter       |
|:-------------------|:-------|:----------|:-------------|:---------------|
|nörmalization       |199 ns  |137 ns     |133 ns        |251 ns (621 ns) |
|No\u0308rmalization |427 ns  |836 ns     |845 ns        |573 ns (948 ns) |

The column with the results for the iterator shows both the measurement with and without initialization of the iterator, which contain buffers that don't need to be reinitialized upon reuse.

As you can see, detecting whether a string is normalized can be quite efficient. A lot of the cost of normalizing in the second row is for the initialization of buffers, the cost of which is amortized when one is processing larger strings. As it turns out, these buffers are rarely needed, so we may change the implementation at some point to speed up the common case for small strings even further.

## Conclusion

If you're dealing with text inside Go, you generally do not have to use the unicode/norm package to normalize your text. The package may still be useful for things like ensuring that strings are normalized before sending them out or to do advanced text manipulation.

This article briefly mentioned the existence of other go.text packages as well as multilingual text processing and it may have raised more questions than it has given answers. The discussion of these topics, however, will have to wait until another day.

By Marcel van Lohuizen

## あわせて読みたい

* [Strings, bytes, runes and characters in Go](https://blog.golang.org/strings)
  * [Goにおける文字列、バイト、ルーンと文字](./strings/)

