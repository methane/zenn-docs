---
title: "go-mysqlでany型へのScan()が使いやすくなります"
emoji: "🕌"
type: "tech"
topics: ["MySQL", "Go"]
published: true
---

`github.com/go-sql-driver/mysql` (以降 go-mysql) の v1.8 から、Scanの振る舞いが変わります。

go-mysql v1.7.1 までは、次のコードの2つのクエリのScan結果は異なっていました。

```go
// db の接続設定で interpolateParams が false の場合
var v any
db.QueryRow("SELECT 123 WHERE ? = 1", 1).Scan(&v)
fmt.Printf("%T %v\n", v, v)  // int64 123

db.QueryRow("SELECT 123 WHERE 1 = 1").Scan(&v)
fmt.Printf("%T %v\n", v, v)  // []uint8 [49 50 51]
```

go-mysql v1.8 (未リリース)からは、これが上の動作に統一されます。([プルリクエスト](https://github.com/go-sql-driver/mysql/pull/1452))

この振る舞いの違いは、独自の型に[Scannerインターフェイス](https://pkg.go.dev/database/sql#Scanner)を実装するときにも影響していました。 `Scan(src any)` のsrcには `nil, int64, float64, bool, []byte, string, time.Time` のいずれかの型の値がくるという定義になっているのですが、ここにもDB上の整数型の値が `int64` で入力される場合と `[]byte` で入力される場合があり、後者を予期していない場合にバグの原因になっていました。


## 背景

### database/sql の設計

ユーザーがScan()を呼ぶとき、 `database/sql` はドライバーの [`Rows.Next()`](https://pkg.go.dev/database/sql/driver#Rows)を呼び出します。 `Rows.Next()` の定義は次のとおりです。

```go
type Rows interface {
    // (略)

    // Next is called to populate the next row of data into
	// the provided slice. The provided slice will be the same
	// size as the Columns() are wide.
	// (略)
	Next(dest []Value) error
}
```

この `Value` 型の実体(underlaying type)は `any` (`type Value any`)です。ドライバが提供する `Rows.Next()` の実装はこの `dest` にDBから読み込んだ値を書き込んでいくことになります。このときにユーザーがScanに渡した型をドライバは知りません。 `database/sql` が、ドライバが dest に書き込んだ値をScanに渡されたポインターに変換しつつ書き込みます。

例えば、DBから大量のレコードを取得してCSVに出力するような例を考えてみましょう。このとき、ユーザーとして一番効率がいい選択肢は `sql.RawBytes` 型(実体は `[]byte`) を`Scan()`に渡すことです。

* ドライバの受信バッファ内のデータが直接利用可能な場合は、 `sql.RawBytes` が直接その受信バッファ内のデータを参照させられる。
* それ以外の場合、例えば受信データが整数値なら、 `database/sql` が `[]byte` 内に文字列として書き込み、それを `sql.RawBytes` にする。

このことを考えると、ドライバとしては受信バッファの中のデータがそのまま利用可能な場合は `dest` に `[]byte` として渡すのが最善の選択になります。


### MySQLのプロトコル

MySQLプロトコル (新しい X protocol と区別するために client/server プロトコルと呼ぶこともある) には、クエリの結果を受け取るためのプロトコルが2種類あります。テキストプロトコルとバイナリプロトコルです。

テキストプロトコルは、QUERYコマンドでクエリ文字列をサーバーに送信してすぐにそのクエリの結果を受け取るときに使われます。テキストプロトコルでは整数の123は、"123"という文字列で返ってきます。

バイナリプロトコルは、まずPREPAREコマンドでクエリ文字列を送り、次にSTMT_EXECUTEコマンドでそのクエリを実行したときに利用されます。PREPAREコマンドで送るクエリにはプレースホルダ `?` を含められ、そのプレースホルダにバインドする値はEXECUTEコマンドで送ります。バイナリプロトコルでは整数は独自の形式でエンコードされているので、 `database/sql` やユーザーが直接使うことはできません。

そこで go-mysql は、 `db.Query()` 等でパラメータがないときはサーバーとのラウンドトリップを減らすためにQUERYコマンドを使い、パラメータがあるときはPREPARE,STMT_EXECUTEコマンドと、PREPAREしたクエリをもう使わないことを伝えるためのSTMT_CLOSEコマンドの合計3ラウンドトリップを使ってクエリを実行します。

このため、クエリパラメータのありなしでテキストプロトコルを使うかバイナリプロトコルを使うかが切り替わり、 `database/sql` に文字列形式([]byte)で返すか整数(int64)で返すかの違いが発生していました。

ちなみに、プレースホルダにパラメータをバインドする処理をドライバ内で実行するオプション `interpolateParams=true` を使えばパラメータがあってもテキストプロトコルを使えますし、逆にパラメータがなくても `db.Prepare()` を使えばバイナリプロトコルを使えるので、この仕組みを知っていれば go-mysql v1.7.1 まででも出力を統一することができます。

## 改善した経緯

クエリの結果が何の型になるか事前に知らないプログラムを書く場合に、Scan()にany型を渡すのは自然なアイデアです。[^1] そのため、クエリパラメータの有無でany型に入る型と値が変化する罠に引っかかる人が定期的に発生します。

さらにScannerインターフェイスを実装する時の罠に同僚が引っかかった事もあり、改善方法について真剣に考えることにしました。

* DATETIME型については、別の理由ですでにオプションがあるので問題にならない。
* 文字列とバイト列を区別するために `[]byte` ではなく `string` を返すのは、データを格納するメモリのアロケーションとデータのコピーが必要になり、デメリットが大きい。さらに `sql.RawBytes` の効果を大きく損ねる。
* 整数型、float型については、 `dest[i]` (any型) に `[]byte` を格納する場合も `int64`, `float64` を格納する場合もアロケーションの回数は変わらず、サイズはむしろ小さくなる。 (スライスはptr,len,capで構成されるのでintの3つ分のサイズ)

	* Scan先が `sql.RawBytes` だった場合、文字列→整数→文字列変換と、追加のアロケーション1回が発生するが、そのコストは長さが分からない文字列と違って許容しやすい。

これらを考慮した上で、メンテナンスコストが高くなるオプション追加ではなく、常に int, float を変換する方向で行くことにしました。これだけでも any へのScanが大分楽になるはずです。

`sql.RawBytes` を使って大量データをCSVに変換するようなプログラムを書くときは、 v1.7.1 まではテキストプロトコルの方が効率が良かったかもしれませんが、 v1.8 からはバイナリプロトコルを使った方が桁数の大きい整数、実数の文字列→int/float変換が不要になって効率が良くなるかもしれません。

[^1]: 一応、 [`Rows.ColumnTypeScanType()`](https://pkg.go.dev/database/sql/driver#RowsColumnTypeScanType) や [`Rows.ColumnTypeDatabaseTypeName()`](https://pkg.go.dev/database/sql/driver#RowsColumnTypeDatabaseTypeName) を使えば実行時にクエリの出力する型を取得してanyではなく適切な型をScanに渡すことは可能ですが、多くの人が気付きませんし、リフレクションを使うのでなかなか面倒なコードになってしまいます。
