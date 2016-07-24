+++
date = "2014-03-13T08:19:15+09:00"
draft = false
title = "Goの並行パターン：パイプラインとキャンセル (Go Concurrency Patterns: Pipelines and cancellation)"
tags = ["concurrency", "pipeline", "cancellation"]
+++

# Goの並行パターン：パイプラインとキャンセル
[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines) by Sameer Ajmani

## はじめに
Goの並行性に関する基本要素によって、I/Oや複数のCPIを効率的に使うことができるストリーミングデータパイプラインを
簡単に構築することができます。この記事ではそのようなパイプラインの例を紹介し、操作が失敗したときに発生する
繊細な事柄にハイライトを当て、また失敗に綺麗に対応するテクニックを紹介します。

## パイプラインとはなにか
Goにおいて、パイプラインの厳密な定義はありません。パイプラインは数ある並行プログラミングの種類の一つに過ぎません。
正式な定義ではないですが、パイプラインとはチャンネルによって接続された一連の _ステージ_ を挿します。
そこでは、各ステージでは同じ関数を実行するゴルーチンのまとまりになっています。
各ステージではゴルーチンは次の役割を果たします。

* _上流_ から _流入_ チャンネル経由で値を受け取る
* そのデータに対してある関数を実行し、通常は新しい値を生成する
* _下流_ へ _流出_ チャンネル経由で値を送信する

各ステージでは、任意の数の流入と流出のチャンネルを持っています。ただし最初と最後のステージは例外で、
それぞれ流出と流入のチャンネルのみが存在します。最初のステージは時々 _ソース_ あるいは _プロデューサー_ と呼ばれ、
最後のステージは _シンク_ あるいは _コンシューマー_ と呼ばれます。

パイプラインの考え方とそのテクニックを説明するために単純なパイプラインの例から始めてみましょう。
あとでより現実的な例を紹介します。

## 数字を平方する
3つのステージからなるパイプラインを考えてみましょう。

最初のステージ `gen` は整数のリストからチャンネルに変換する関数です。このチャンネルがリスト内の整数を出すことになります。
`gen` 関数は整数をチャンネルに送信するゴルーチンを起動し、すべての値が送信されたらチャンネルを閉じます。

```
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}
```

2番めのステージは `sq` で、チャンネルから整数を受信して、受信した整数それぞれの平方を出すチャンネルを返します。
流入のチャンネルが閉じて、すべての値を下流に送った後に、流出のチャンネルを閉じます。

```
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

`main` 関数はパイプラインを設定し、最後のステージを実行します。2番めのステージから値を受信し、チャンネルが閉じるまで、
それぞれを出力します。

```
func main() {
    // パイプラインを設定する。
    c := gen(2, 3)
    out := sq(c)

    // 出力を消費する。
    fmt.Println(<-out) // 4
    fmt.Println(<-out) // 9
}
```

`sq` では流入と流出のチャンネルでそれぞれ同じ型なので、 `sq` を何回でも繰り返すことができます。
また `main` を他のステージと同様にrangeを使ったループに書き換えることもできます。

```
func main() {
    // パイプラインを設定して出力を消費する。
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16 then 81
    }
}
```

## ファンアウト、ファンイン
チャンネルが閉じるまで複数の関数が1つのチャンネルを読み込むことが可能です。
これは _ファンアウト_ と呼ばれます。この構成はCPU使用率とI/Oを平行に使うために
ワーカー群に仕事を分配する方法を提供しています。

1つの関数は入力チャンネルを多重化して1つのチャンネルに流し込むことで、すべての入力が閉じるまで
複数の入力から読み込み処理をすることができて、流し込む先のチャンネルはすべての入力が閉じると閉じられます。
これを _ファンイン_ と呼びます。

先ほどのパイプラインを2つの `sq` のインスタンスを実行するように変更し、それぞれが同一の
入力チャンネルから読み込むようにできます。ここで新しい関数 `merge` を用意して結果をファンインします。

```
func main() {
    in := gen(2, 3)

    // sq の仕事を同一のチャンネル in から読み込む2つのゴルーチンに分配します。
    c1 := sq(in)
    c2 := sq(in)

    // c1 と c2 の結果をマージしたものを消費します。
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4 then 9, or 9 then 4
    }
}
```

`merge` 関数は、流入チャンネルそれぞれに対してゴルーチンを起動して、値を唯一の流出チャンネルに
コピーすることで、チャンネルのリストを1つのチャンネルに変換します。すべての `output` ゴルーチンが
起動したら、 `merge` は更にもう一つゴルーチンを起動して、そのチャンネルへの送信がすべて終わったら
流出チャンネルを閉じます。

閉じたチャンネルに送信するとパニックになるので、closeを呼ぶ前にすべての値が送信されていることを
確実にすることが大事です。 [sync.WaitGroup](http://golang.org/pkg/sync/#WaitGroup) 型はこのような
同期を用意する簡単な方法を提供しています。

```
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // cs 内の各入力チャンネルに対して output ゴルーチンを起動。
    // output は c が閉じるまで c から out に値をコピーして、その後 wg.Done を呼び出す。
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // output ゴルーチンがすべて終了したら out を閉じるためのゴルーチンを起動する。
    // これは wg.Add が呼び出された後に起動しなければならない。
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

## 早めに止める
私たちのパイプライン関数にはパターンがあります。

* 送信が全て完了したら流出チャンネルを閉じるステージ
* チャンネルが閉じるまで流入チャンネルから値を受信し続けるステージ

このパターンは受信の各ステージを `range` のループで書くことができ、またすべての値が無事に下流に送信されたら
確実にすべてのゴルーチンが終了してくれます。

しかし実際のパイプラインでは、ステージで流入するすべての値を受信するとは限りません。
ときには設計がすべてを受信しないようにしていることがあります。受信するステージでは
処理のためにすべての値の中の一部だけが必要なことがあります。また、しばしば流入する値が
前のステージでの異常値を表している場合に終了することがあります。
どちらの場合でも、受信するステージでは残りの値を待つべきではないですし、前段のステージで
後段のステージで必要にならない値を生成するのを止めたいものです。

私たち例のパイプラインでは、ステージが流入する値のすべてを消費できなかった場合、
それらの値を送信しようとするゴルーチンはいつまでもブロックし続けます。

```
    // 出力からの最初の値を消費する
    out := merge(c1, c2)
    fmt.Println(<-out) // 4 か 9
    return
    // outから2番めの値を受け取ってないため
    // 出力用のゴルーチンは2番めの値を送信しようとしてとどまってしまいます
}
```

これがリソースリークです。ゴルーチンがメモリとランタイムの資源を消費して、ゴルーチンのスタック内の
ヒープの参照はデータがガベージコレクトされないようにします。ゴルーチンはガベージコレクトされません。
ゴルーチンは自分自身で終了しなければいけません。

パイプラインの下流のステージがすべての流入する値を受信できなかった場合にも上流のステージが終了するように
変更する必要があります。修正方法の一つとして、流出のチャンネルにバッファを持たせる方法があります。
バッファは決まった数の値を持つことができます。バッファ内に空きができ次第送信の操作が完了します。

```
c := make(chan int, 2) // バッファのサイズは 2
c <- 1  // ただちに成功
c <- 2  // ただちに成功
c <- 3  // 他のゴルーチンが <-c として 1 を受信するまでブロック
```

チャンネルを作成するときに送信される値の数がわかっている場合、バッファの確保によってコードを短くできます。
たとえば、 `gen` を整数のリストをバッファ付きのチャンネルにコピーするようにコードを書き換えて、
新しいゴルーチンを生成しないようにすることができます。

```
func gen(nums ...int) <-chan int {
    out := make(chan int, len(nums))
    for _, n := range nums {
        out <- n
    }
    close(out)
    return out
}
```

先ほどのパイプラインの話に戻ると、 `merge` によって返される流出チャンネルにバッファを追加することを考えてみましょう。

```
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int, 1) // 未読の入力に対して十分な領域
    // ... あとはさきほどと同じ ...
```

この変更でプログラム中でゴルーチンがブロックしてしまう件は修正されましたが、これは良くないコードです。
ここでバッファサイズを1にしているのは、 `merge` が受信する値の数と下流のステージで消費される値の数を知っているから
できることです。これは脆い設計です。 `gen` にさらに値を渡した場合、あるいは下流で読み取る値を減らした場合、
再びブロックするゴルーチンが発生してしまいます。

かわりに、下流のステージがこれ以上入力を受け付けないことを送信元に伝える方法を提供する必要があります。

## 明示的なキャンセル
`main` が `out` からの値をすべて受信せずに終了すると決めたとき、 `main` は上流のステージのゴルーチンに
送信しようとしている値を破棄するように伝えなければいけません。これは `done` というチャンネルに値を送ることで
実現しています。 `main` は潜在的にブロックする可能性がある2つの送信元があるので2つ値を送信します。

```
func main() {
    in := gen(2, 3)

    // in からともに値を読み取る2つのゴルーチンに sq の処理を分配します
    c1 := sq(in)
    c2 := sq(in)

    // 最初の値を出力から消費します
    done := make(chan struct{}, 2)
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // まだ処理を続けている送信元に終了することを伝えます
    done <- struct{}{}
    done <- struct{}{}
}
```

送信するゴルーチンでは、送信の操作を `select` 文に置き換えて、 `out` への送信があった場合、もしくは `done`
から値を受信した場合に処理が進むようにします。 `done` の型は空の構造体です。その理由は、値は関係ないからです。
つまり、単純に `out` への送信を辞めるべきタイミングを示すイベントを受信するだけのものだからです。
`output` ゴルーチンは上流のステージがブロックされないように流入チャンネルの `c` に対してループを続けます。
（すぐ後で、このループが早めに終われるようにするかをお話します）

```
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // output ゴルーチンを cs 内の各入力チャンネルに対して起動します。
    // output は c がチャンネルを閉じるまで、あるいは、doneから値を受け取るまで
    // 値をコピーし続け、その後 wg.Done を呼び出します。
    output := func(c <-chan int) {
        for n := range c {
            select {
            case out <- n:
            case <-done:
            }
        }
        wg.Done()
    }
    // ... あとはさきほどと同じ ...
```

このアプローチには問題が有ります。下流の _各_ レシーバーは潜在的にブロックしてしまう可能性のある上流の
送信元の数を事前に知る必要があり、それらの送信元が早く終了するためのシグナルを用意する必要があります。
これらの数を数えているのは退屈でエラーの温床となります。

未知数で制限のない数のゴルーチンに対して下流に値を送信するのを止めさせる方法が必要です。
Goではチャンネルを閉じることでこれを実現できます。なぜならば、閉じたチャンネルに対しての
受信操作は直ちに実行されチャンネルの要素の型のゼロ値が返されるからです。（ [参照](http://golang.org/ref/spec#Receive_operator) )

これはつまり、 `main` ですべての送信元を単純に `done` チャンネルを閉じることでブロックから解放できる
ということを意味しています。この閉じる操作によって送信元に効率的にシグナルを配信することができます。
私たちのパイプラインの _各_ 関数が `done` を引数として受け取るように拡張して、 `defer` 文によって
`done` が閉じられるようにし、 `main` 内のすべての終了処理からパイプラインの各ステージに終了するようにシグナルを送信します。

```
func main() {
    // パイプライン全体で共有される done チャンネルを設定し、このパイプラインが
    // 終了するときにチャンネルを閉じます。これは起動したすべてのゴルーチンが
    // 終了するためのシグナルとしてのものです。
    done := make(chan struct{})
    defer close(done)

    in := gen(done, 2, 3)

        // in から値を読み取る2つのゴルーチンに sq の処理を分散する。
    c1 := sq(done, in)
    c2 := sq(done, in)

    // output から最初の1つの値を消費する。
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // deferされた呼び出しによって done は閉じられる
}
```

これでパイプラインでの各ステージは `done` が閉じられるとすぐに終了できるようになりました。
上流の送信元である `sq` は `done` が閉じられれば送信をやめるとわかっているので、
`merge` 内の `output` ルーチンは流入チャンネルの値をすべて出すことなく終了処理をすることが出来ます。
`output` は `defer` 文によって、すべての終了処理経路で `wg.Done` が呼ばれることを保証しています。

```
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // cs 内の各入力チャンネルに対して output ゴルーチンを起動する。
    // output は c 内の値を c もしくは done が閉じられるまで out にコピーする。
    // その後 wg.Done を呼ぶ。
    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }
    // ... あとはさきほどと同じ ...
```

同様に、 `sq` も `done` が閉じられるとすぐに終了処理を行うことができます。 `sq` は `defer` 文によって
すべての終了処理経路の中で `out` チャンネルが閉じられることを保証しています。

```
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

次がパイプラインを構築する際のガイドラインです。

* ステージはすべての送信の操作が終わったときに出力チャンネルを閉じます。
* ステージは入力チャンネルが閉じるまで値を受信し続けるか、さもなくば送信側を解放します。

パイプラインは値を取得する十分バッファがあることを確認できる場合、もしくは受信側がチャンネルを放棄した時に
明示的に送信側にシグナルを送れるようにすることで、送信者を解放します。

## ディレクトリツリーをダイジェストする

より現実的なパイプラインを考えてみましょう。

MD5はファイルのチェックサムとして便利なメッセージダイジェストアルゴリズムです。
コマンドラインツールの `md5sum` はファイル一覧のダイジェスト値を表示します。 

```
% md5sum *.go
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

私たちのサンプルプログラムは `md5sum` に似ていますが、ディレクトリを引数に取り、その配下にあるファイルをパス順に並べ、
それぞれのダイジェスト値を表示します。

```
% go run serial.go .
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

`main` 関数は、パス名とダイジェスト値のマップを返す `MD5All` というヘルパー関数を呼び出し、それをソートして結果を表示します。

```
func main() {
    // 指定されたディレクトリ配下のすべてのファイルのMD5チェックサムを計算し、
    // パス名順に結果を並べて表示する。
    m, err := MD5All(os.Args[1])
    if err != nil {
        fmt.Println(err)
        return
    }
    var paths []string
    for path := range m {
        paths = append(paths, path)
    }
    sort.Strings(paths)
    for _, path := range paths {
        fmt.Printf("%x  %s\n", m[path], path)
    }
}
```

`MD5All` 関数が議論の対象です。　`serial.go` では、並行性をまったく使わずに、
ディレクトリを再帰探索しながら単順に個々のファイルを読み込んで、チェックサムを計算しています。

```
// MD5All は root 配下のすべてのファイルを読み込み、各ファイルのファイルパスとMD5チェックサムの
// マップを返します。ディレクトリの再帰探索の失敗、あるいは読み込みの失敗がが発生したら
// MD5Allはエラーを返します。
func MD5All(root string) (map[string][md5.Size]byte, error) {
    m := make(map[string][md5.Size]byte)
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if !info.Mode().IsRegular() {
            return nil
        }
        data, err := ioutil.ReadFile(path)
        if err != nil {
            return err
        }
        m[path] = md5.Sum(data)
        return nil
    })
    if err != nil {
        return nil, err
    }
    return m, nil
}
```

## 並列ダイジェスト

[parallel.go](https://blog.golang.org/pipelines/parallel.go)　では `MD5All` を2段階のパイプラインに分けています。
第1段階である `sumFiles` では、ディレクトリを探索し個々のファイルをそれぞれのゴルーチンでダイジェストし、
`result` 型の値のチャンネルに結果を送ります。

```
type result struct {
    path string
    sum  [md5.Size]byte
    err  error
}
```

`sumFiles` は2つのチャンネルを返します。1つは結果を表すもの、もう1つは `filepath.Walk` によって返されるエラーです。
ディレクトリ探索の関数は個々のファイルを処理する新しいゴルーチンを立ち上げ、 `done` を確認します。
`done` が閉じられたら、探索を直ちに終了します。

```
func sumFiles(done <-chan struct{}, root string) (<-chan result, <-chan error) {
    // For each regular file, start a goroutine that sums the file and sends
    // the result on c.  Send the result of the walk on errc.
    c := make(chan result)
    errc := make(chan error, 1)
    go func() {
        var wg sync.WaitGroup
        err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            wg.Add(1)
            go func() {
                data, err := ioutil.ReadFile(path)
                select {
                case c <- result{path, md5.Sum(data), err}:
                case <-done:
                }
                wg.Done()
            }()
            // Abort the walk if done is closed.
            select {
            case <-done:
                return errors.New("walk canceled")
            default:
                return nil
            }
        })
        // Walk has returned, so all calls to wg.Add are done.  Start a
        // goroutine to close c once all the sends are done.
        go func() {
            wg.Wait()
            close(c)
        }()
        // No select needed here, since errc is buffered.
        errc <- err
    }()
    return c, errc
}
```

`MD5All` は `c` からダイジェスト値を受け取ります。
`MD5All` はエラーがあれば先にエラーを返し、 `defer` で `done` を閉じます。

```
func MD5All(root string) (map[string][md5.Size]byte, error) {
    // MD5All closes the done channel when it returns; it may do so before
    // receiving all the values from c and errc.
    done := make(chan struct{})
    defer close(done)

    c, errc := sumFiles(done, root)

    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }
    if err := <-errc; err != nil {
        return nil, err
    }
    return m, nil
}
```

## 限定的並列処理

[parallel.go](https://blog.golang.org/pipelines/parallel.go) での `MD5All` の実装では各ファイルに対し新しいゴルーチンを
起動させています。多くのファイルがあるディレクトリでは、この実装だとマシンに搭載されている以上のメモリをアロケートしかねません。

並列処理の中で読み込まれるファイル数を限定することで、このアロケーションを制限することができます。
[bounded.go](https://blog.golang.org/pipelines/bounded.go) では、ファイルの読み込みに決められた数のゴルーチンを
作成することで実現しています。今回のパイプラインは3段階になっています。ディレクトリの再帰探索、ファイルの読み込みとダイジェスト値の計算、
そしてダイジェスト値の回収です。

第1段階の `walkFiles` は、ファイルツリー内のファイルのパスを返します。

```
func walkFiles(done <-chan struct{}, root string) (<-chan string, <-chan error) {
    paths := make(chan string)
    errc := make(chan error, 1)
    go func() {
        // Walkが終了したらpathsチャンネルを閉じます。
        defer close(paths)
        // errcはバッファ済みなのでselectは必要ありません。
        errc <- filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            select {
            case paths <- path:
            case <-done:
                return errors.New("walk canceled")
            }
            return nil
        })
    }()
    return paths, errc
}
```

第2段階では一定数のダイジェスト値を計算するゴルーチンを起動します。このゴルーチンはパスにあるファイル名を受け取り、
結果をチャンネル `c` に送ります。

```
func digester(done <-chan struct{}, paths <-chan string, c chan<- result) {
    for path := range paths {
        data, err := ioutil.ReadFile(path)
        select {
        case c <- result{path, md5.Sum(data), err}:
        case <-done:
            return
        }
    }
}
```

先の例と違って、ダイジェスト値計算用ゴルーチンは出力チャンネルを閉じません。その理由は複数のゴルーチンが1つの共通のチャンネルに
値を送っているからです。代わりに、 `MD5All` 内ですべてのダイジェスト値計算用ゴルーチンが終了したらチャンネルを閉じるようにしています。

```
    // ファイルの読み込みとダイジェスト値の計算をするゴルーチンを一定数起動
    c := make(chan result)
    var wg sync.WaitGroup
    const numDigesters = 20
    wg.Add(numDigesters)
    for i := 0; i < numDigesters; i++ {
        go func() {
            digester(done, paths, c)
            wg.Done()
        }()
    }
    go func() {
        wg.Wait()
        close(c)
    }()
```

他の方法として、ダイジェスト計算用ゴルーチンそれぞれが出力用チャンネルを作ってそれを返すことも可能ですが、
その場合は結果をファンインさせるゴルーチンが追加で必要になります。

第3段階では `c` から結果をすべて受け取り、そのあと `errc` 内のエラーを確認します。
この確認は `c` の結果を受け取った後でなければいけません。なぜならこれより前だと、 `walkFiles` が下流に値を送るのをブロックしてしまうからです。

```
    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }
    // Walkが失敗したかを確認
    if err := <-errc; err != nil {
        return nil, err
    }
    return m, nil
}
```

## 結論

この記事ではGoでストリーミングデータパイプラインを構築するテクニックを紹介しました。
このようなパイプラインでエラーを扱う場合、各段階においてパイプラインが下流に値を送信するのをブロックしないように、
そして下流の段階では入力データについて心配する必要がなくなるように、注意しなければいけません。
本記事の例では、 `"done"` シグナルをパイプラインによって起動されたすべてのゴルーチンに配信しする方法をお見せし、
またパイプラインを正しく構築するガイドラインを定義しました。

より深く理解したい人には次の記事をおすすめします。

* [Go Concurrency Patterns](http://talks.golang.org/2012/concurrency.slide#1) ([動画](https://www.youtube.com/watch?v=f6kdp27TYZs)) ではGoにおける並行プログラミングの基礎とその適用方法をいくつか紹介しています。
* [Advanced Go Concurrency Patterns](http://blog.golang.org/advanced-go-concurrency-patterns) ([動画](http://www.youtube.com/watch?v=QDDwwePbDtw)) ではより複雑なGoの機能、特にselectについて触れています。
* Douglas McIlroy の論文 [Squinting at Power Series](http://swtch.com/~rsc/thread/squint.pdf) では、
Goのような並行性がどのように複雑な計算を華麗にサポートするかを説明しています。

By Sameer Ajmani

## あわせて読みたい
* [Go Concurrency Patterns: Context](https://blog.golang.org/context)
  * [Goの並行パターン：コンテキスト (Go Concurrency Pattern: Context)](../context/)
* [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
* [Advanced Go Concurrency Patterns](https://blog.golang.org/advanced-go-concurrency-patterns)
* [Concurrency is not parallelism](https://blog.golang.org/concurrency-is-not-parallelism)
* [Go videos from Google I/O 2012](https://blog.golang.org/go-videos-from-google-io-2012)
* [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)
  * [Goの並行パターン：タイムアウトと進行](../go-concurrency-patterns-timing-out-and/)
* [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)