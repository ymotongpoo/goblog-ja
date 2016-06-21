+++
date = "2013-10-23T09:18:55+09:00"
draft = true
title = "strings"
tags = ["strings", "bytes", "runes", "characters"]
+++

# Goにおける文字列、バイト、ルーンと文字 (Strings, bytes, runes and characters)
[Strings, bytes, runes and characters in Go](https://blog.golang.org/strings) By Rob Pike

## はじめに

[1つ前の記事](./slices/) では、その実装の背後にある機能を解説する例とともに、Goにおいてスライスがどのように動作するかを説明しました。
その知識を前提として、この記事ではGoにおける文字列について話します。
まず最初に、文字列はブログの記事にしては簡単すぎるように見えるかもしれませんが、上手に使うには文字列の動作を理解するだけでなく、
バイト、文字、ルーンの違いについても理解し、UnicodeとUTF-8の違いについても理解し、文字列と文字列リテラルの違いについても理解し、
その他多くの細かな違いについて理解する必要があります。

この話題を議論するときの1つのアプローチとして、FAQである「Goの文字列のn番目のインデックスにアクセスした時に、
なぜn番目の文字を取得できないのか」という質問の回答を考えてみましょう。この記事で説明していきますが、この質問には
現代社会でテキストがどのように動作しているかを多くの観点から考えるきっかけとなります。

Goにかぎらず、これらの問題について考える最高の導入は、Joel Spolskyの有名なブログポスト、[The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html) です。
彼がその記事の中で挙げた点を、この記事でも繰り返し言及します。

## stringとは何か

基礎からはじめましょう。

Goでは、文字列は実際には読み取り専用のバイトのスライスでした。バイトのスライスがなにかについて不確かな場合、あるいはそれがどう動作するか
不確かな場合は、[前のブログポスト](./slices/) を読んで下さい。この記事ではすでに前のブログポストを読んでいることを前提とします。

stringは *任意の* バイトを保持できることをはっきりと述べておくことは重要です。
stringはUnicode文字もUTF-8文字も、その他の事前定義の形式の文字を持つ必要はありません。
stringの中身を考える限りにおいては、それはバイトのスライスについて考えることと同義です。

（すぐあとで説明しますが）あるバイト値を定数で持つように `\xNN` という記法を使って文字列リテラルを定義しました。
（もちろん、16進数でのバイト値の範囲は両端含んで `00` から `FF` です）

```
    const sample = "\xbd\xb2\x3d\xbc\x20\xe2\x8c\x98"
```

## Printing strings

Because some of the bytes in our sample string are not valid ASCII, not even valid UTF-8, printing the string directly will produce ugly output. The simple print statement

```
    fmt.Println(sample)
```

produces this mess (whose exact appearance varies with the environment):

```
��=� ⌘
```

To find out what that string really holds, we need to take it apart and examine the pieces. There are several ways to do this. The most obvious is to loop over its contents and pull out the bytes individually, as in this for loop:

```
    for i := 0; i < len(sample); i++ {
        fmt.Printf("%x ", sample[i])
    }
```

As implied up front, indexing a string accesses individual bytes, not characters. We'll return to that topic in detail below. For now, let's stick with just the bytes. This is the output from the byte-by-byte loop:

```
bd b2 3d bc 20 e2 8c 98
```

Notice how the individual bytes match the hexadecimal escapes that defined the string.

A shorter way to generate presentable output for a messy string is to use the `%x` (hexadecimal) format verb of fmt.Printf. It just dumps out the sequential bytes of the string as hexadecimal digits, two per byte.

```
    fmt.Printf("%x\n", sample)
```

Compare its output to that above:

```
bdb23dbc20e28c98
```

A nice trick is to use the "space" flag in that format, putting a space between the % and the x. Compare the format string used here to the one above,

```
    fmt.Printf("% x\n", sample)
```

and notice how the bytes come out with spaces between, making the result a little less imposing:

```
bd b2 3d bc 20 e2 8c 98
```

There's more. The %q (quoted) verb will escape any non-printable byte sequences in a string so the output is unambiguous.

```
    fmt.Printf("%q\n", sample)
```

This technique is handy when much of the string is intelligible as text but there are peculiarities to root out; it produces:

```
"\xbd\xb2=\xbc ⌘"
```

If we squint at that, we can see that buried in the noise is one ASCII equals sign, along with a regular space, and at the end appears the well-known Swedish "Place of Interest" symbol. That symbol has Unicode value U+2318, encoded as UTF-8 by the bytes after the space (hex value 20): e2 8c 98.

If we are unfamiliar or confused by strange values in the string, we can use the "plus" flag to the %q verb. This flag causes the output to escape not only non-printable sequences, but also any non-ASCII bytes, all while interpreting UTF-8. The result is that it exposes the Unicode values of properly formatted UTF-8 that represents non-ASCII data in the string:

```
    fmt.Printf("%+q\n", sample)
```

With that format, the Unicode value of the Swedish symbol shows up as a \u escape:

```
"\xbd\xb2=\xbc \u2318"
```

These printing techiques are good to know when debugging the contents of strings, and will be handy in the discussion that follows. It's worth pointing out as well that all these methods behave exactly the same for byte slices as they do for strings.

Here's the full set of printing options we've listed, presented as a complete program you can run (and edit) right in the browser:

```
package main

import "fmt"

func main() {
    const sample = "\xbd\xb2\x3d\xbc\x20\xe2\x8c\x98"

    fmt.Println("Println:")
    fmt.Println(sample)

    fmt.Println("Byte loop:")
    for i := 0; i < len(sample); i++ {
        fmt.Printf("%x ", sample[i])
    }
    fmt.Printf("\n")

    fmt.Println("Printf with %x:")
    fmt.Printf("%x\n", sample)

    fmt.Println("Printf with % x:")
    fmt.Printf("% x\n", sample)

    fmt.Println("Printf with %q:")
    fmt.Printf("%q\n", sample)

    fmt.Println("Printf with %+q:")
    fmt.Printf("%+q\n", sample)
}
```

[Exercise: Modify the examples above to use a slice of bytes instead of a string. Hint: Use a conversion to create the slice.]

[Exercise: Loop over the string using the %q format on each byte. What does the output tell you?]

## UTF-8 and string literals

As we saw, indexing a string yields its bytes, not its characters: a string is just a bunch of bytes. That means that when we store a character value in a string, we store its byte-at-a-time representation. Let's look at a more controlled example to see how that happens.

Here's a simple program that prints a string constant with a single character three different ways, once as a plain string, once as an ASCII-only quoted string, and once as individual bytes in hexadecimal. To avoid any confusion, we create a "raw string", enclosed by back quotes, so it can contain only literal text. (Regular strings, enclosed by double quotes, can contain escape sequences as we showed above.)

```
func main() {
    const placeOfInterest = `⌘`

    fmt.Printf("plain string: ")
    fmt.Printf("%s", placeOfInterest)
    fmt.Printf("\n")

    fmt.Printf("quoted string: ")
    fmt.Printf("%+q", placeOfInterest)
    fmt.Printf("\n")

    fmt.Printf("hex bytes: ")
    for i := 0; i < len(placeOfInterest); i++ {
        fmt.Printf("%x ", placeOfInterest[i])
    }
    fmt.Printf("\n")
}
```

The output is:

```
plain string: ⌘
quoted string: "\u2318"
hex bytes: e2 8c 98
```

which reminds us that the Unicode character value U+2318, the "Place of Interest" symbol ⌘, is represented by the bytes e2 8c 98, and that those bytes are the UTF-8 encoding of the hexadecimal value 2318.

It may be obvious or it may be subtle, depending on your familiarity with UTF-8, but it's worth taking a moment to explain how the UTF-8 representation of the string was created. The simple fact is: it was created when the source code was written.

Source code in Go is defined to be UTF-8 text; no other representation is allowed. That implies that when, in the source code, we write the text

```
`⌘`
```

the text editor used to create the program places the UTF-8 encoding of the symbol ⌘ into the source text. When we print out the hexadecimal bytes, we're just dumping the data the editor placed in the file.

In short, Go source code is UTF-8, *so the source code for the string literal is UTF-8 text*. If that string literal contains no escape sequences, which a raw string cannot, the constructed string will hold exactly the source text between the quotes. Thus by definition and by construction the raw string will always contain a valid UTF-8 representation of its contents. Similarly, unless it contains UTF-8-breaking escapes like those from the previous section, a regular string literal will also always contain valid UTF-8.

Some people think Go strings are always UTF-8, but they are not: only string literals are UTF-8. As we showed in the previous section, string values can contain arbitrary bytes; as we showed in this one, string literals always contain UTF-8 text as long as they have no byte-level escapes.

To summarize, strings can contain arbitrary bytes, but when constructed from string literals, those bytes are (almost always) UTF-8.

## Code points, characters, and runes

We've been very careful so far in how we use the words "byte" and "character". That's partly because strings hold bytes, and partly because the idea of "character" is a little hard to define. The Unicode standard uses the term "code point" to refer to the item represented by a single value. The code point U+2318, with hexadecimal value 2318, represents the symbol ⌘. (For lots more information about that code point, see its Unicode page.)

To pick a more prosaic example, the Unicode code point U+0061 is the lower case Latin letter 'A': a.

But what about the lower case grave-accented letter 'A', à? That's a character, and it's also a code point (U+00E0), but it has other representations. For example we can use the "combining" grave accent code point, U+0300, and attach it to the lower case letter a, U+0061, to create the same character à. In general, a character may be represented by a number of different sequences of code points, and therefore different sequences of UTF-8 bytes.

The concept of character in computing is therefore ambiguous, or at least confusing, so we use it with care. To make things dependable, there are normalization techniques that guarantee that a given character is always represented by the same code points, but that subject takes us too far off the topic for now. A later blog post will explain how the Go libraries address normalization.

"Code point" is a bit of a mouthful, so Go introduces a shorter term for the concept: rune. The term appears in the libraries and source code, and means exactly the same as "code point", with one interesting addition.

The Go language defines the word rune as an alias for the type int32, so programs can be clear when an integer value represents a code point. Moreover, what you might think of as a character constant is called a rune constant in Go. The type and value of the expression

```
'⌘'
```

is rune with integer value `0x2318`.

To summarize, here are the salient points:

* Go source code is always UTF-8.
* A string holds arbitrary bytes.
* A string literal, absent byte-level escapes, always holds valid UTF-8 sequences.
* Those sequences represent Unicode code points, called runes.
* No guarantee is made in Go that characters in strings are normalized.

## Range loops

Besides the axiomatic detail that Go source code is UTF-8, there's really only one way that Go treats UTF-8 specially, and that is when using a for range loop on a string.

We've seen what happens with a regular for loop. A for range loop, by contrast, decodes one UTF-8-encoded rune on each iteration. Each time around the loop, the index of the loop is the starting position of the current rune, measured in bytes, and the code point is its value. Here's an example using yet another handy Printf format, %#U, which shows the code point's Unicode value and its printed representation:

```
    const nihongo = "日本語"
    for index, runeValue := range nihongo {
        fmt.Printf("%#U starts at byte position %d\n", runeValue, index)
    }
```

The output shows how each code point occupies multiple bytes:

```
U+65E5 '日' starts at byte position 0
U+672C '本' starts at byte position 3
U+8A9E '語' starts at byte position 6
```

[Exercise: Put an invalid UTF-8 byte sequence into the string. (How?) What happens to the iterations of the loop?]

## Libraries

Go's standard library provides strong support for interpreting UTF-8 text. If a for range loop isn't sufficient for your purposes, chances are the facility you need is provided by a package in the library.

The most important such package is [unicode/utf8](http://golang.org/pkg/unicode/utf8/), which contains helper routines to validate, disassemble, and reassemble UTF-8 strings. Here is a program equivalent to the for range example above, but using the DecodeRuneInString function from that package to do the work. The return values from the function are the rune and its width in UTF-8-encoded bytes.

```
    const nihongo = "日本語"
    for i, w := 0, 0; i < len(nihongo); i += w {
        runeValue, width := utf8.DecodeRuneInString(nihongo[i:])
        fmt.Printf("%#U starts at byte position %d\n", runeValue, i)
        w = width
    }
```

Run it to see that it performs the same. The for range loop and DecodeRuneInString are defined to produce exactly the same iteration sequence.

Look at the [documentation](http://golang.org/pkg/unicode/utf8/) for the `unicode/utf8` package to see what other facilities it provides.

## Conclusion

To answer the question posed at the beginning: Strings are built from bytes so indexing them yields bytes, not characters. A string might not even hold characters. In fact, the definition of "character" is ambiguous and it would be a mistake to try to resolve the ambiguity by defining that strings are made of characters.

There's much more to say about Unicode, UTF-8, and the world of multilingual text processing, but it can wait for another post. For now, we hope you have a better understanding of how Go strings behave and that, although they may contain arbitrary bytes, UTF-8 is a central part of their design.

By Rob Pike
