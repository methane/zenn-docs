---
title: "MySQL接続のcollation不整合の原因と対策"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MySQL"]
published: true
---

## まとめ

MySQLの新規接続時のcharset/collation指定は無視されることがあり、エラーや警告にならないので気づきにくいです。

接続直後に `SET NAMES utf8mb4` (collationも指定したい場合は `SET NAMES utf8mb4 COLLATE utf8mb4_bin` など) を実行すると安全です。

## 接続のcollationとデータベースのcollationの不整合

まずは実例を見てみましょう。Ubuntu上で、DockerでMySQL 8.0を動かしつつ、MariaDBのクライアントから接続してみます。

:::details 準備(MySQLを立ち上げてmariadb-clientで接続するまで)
```
# Docker で MySQL を動かす
$ docker pull mysql:latest
$ docker run -d -e MYSQL_ALLOW_EMPTY_PASSWORD=yes -p 3306:3306 --rm --name mysql mysql:latest

# mariadb-client をインストールして接続する。
# 接続時のバナーでクライアントがMariaDB、サーバーがMySQL 8.0.33と確認できる
$ sudo apt-get update
$ sudo apt-get install -y mariadb-client
$ mysql -h 127.0.0.1 -uroot --default-character-set=utf8mb4

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.33 MySQL Community Server - GPL
(snip)
```
:::

```
# デフォルトのcharset/collatinでデータベースとテーブルを作る
MySQL [(none)]> create database test;
Query OK, 1 row affected (0.036 sec)

MySQL [(none)]> use test
Database changed
MySQL [test]> create table test (id integer auto_increment primary key, t varchar(100));
Query OK, 0 rows affected (0.055 sec)

# デフォルトのcollationしか使ってない状況で
# "Illegal mix of collations" エラーを起こしてみる
MySQL [test]> insert into test (t) values ("hello");
Query OK, 1 row affected (0.028 sec)

MySQL [test]> set @v='test';
Query OK, 0 rows affected (0.007 sec)

MySQL [test]> select * from test where t=@v;
ERROR 1267 (HY000): Illegal mix of collations (utf8mb4_0900_ai_ci,IMPLICIT) and (utf8mb4_general_ci,IMPLICIT) for operation '='
```

"Illegal mix of collations" エラーが発生しました。これは、MySQLがクライアントから送られた文字列リテラルに割り当てるcollation(`collation_connection`, これを以降「接続のcollation」と呼びます)と、データベースやテーブルを作成する時に自動で使用するcollation(`collation_server`やcharsetごとのデフォルトのcollation)が異なるためです。

先ほどの例で、接続のcollationと、テーブルが利用しているcollationを確認して見ます。

```
MySQL [test]> select @@collation_connection;
+------------------------+
| @@collation_connection |
+------------------------+
| utf8mb4_general_ci     |
+------------------------+
1 row in set (0.005 sec)

MySQL [test]> show create table test\G
*************************** 1. row ***************************
       Table: test
Create Table: CREATE TABLE `test` (
  `id` int NOT NULL AUTO_INCREMENT,
  `t` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.005 sec)
```

接続のcollationは `utf8mb4_general_ci` で、テーブルのcollationは`utf8mb4_0900_ai_ci`になっていますね。

サーバーはデフォルトのutf8mb4を使っていて、クライアントも `--default-character-set=utf8mb4` を指定しているだけなのに、なぜ不整合が起こるのでしょうか。


## collation不整合が起こる理由

不整合が起こる大きい原因は、MySQL client/server protocolの構造的な欠陥と、MySQLが8.0でutf8mb4のデフォルトのcollationを変更したことにあります。collationの変更についてはたくさん説明があるので、「MySQL 寿司ビール問題」で検索してください。ここではprotocolの方の問題を解説します。

MySQLの接続開始時は、サーバーからハンドシェイクリクエストを送り、クライアントがハンドシェイクレスポンスを返し、サーバーがOKを返す、という3-wayハンドシェイクになっています。クライアントはハンドシェイクレスポンスで collation_id を指定することで、charsetとcollationを指定します。これには2つの構造的欠陥があります。

- 指定したcollationが利用されなくてもエラーにならず、OKパケットからも接続のcollationがわからない。
- クライアントがcharsetのデフォルトのcollationやcollation_idなどのデータを持っている必要があり、それがサーバー側と一致する保証がない。

さきほどのmariadb-clientからMySQLへ接続する例では、mariadb-client に `--default-character-set=utf8mb4` を指定していましたが、mariadbが持っているデータでは utf8mb4 のデフォルトのcollationは `utf8mb4_general_ci` だったので、その collation_id をハンドシェイクレスポンスで送信していました。これがMySQLのutf8mb4のデフォルトのcollationである `utf8mb4_0900_ai_ci` と合わなかったのです。

## collationやcharsetの不整合が起こる組み合わせ

MySQLとMariaDBの違いや、MySQLのバージョンの違いの組み合わせで、以下のような問題が起こります。（ビルド時の設定によるので、必ずしもこの通りになるとは限りません。）

- mysql 5.7 clientに `--default-character-set=utf8mb4` を指定してMySQL 8.0に接続すると、接続のcollationが `utf8mb4_general_ci` になる。（上のMariaDBと同じ）
- mysql 8.0 clientからMySQL 5.7に接続すると、サーバーが `utf8mb4_0900_ai_ci` を知らないので、デフォルトの `latin1_swedish_ci` にfallbackする。
- mysql 8.0 clientからMariaDB 10.xに接続すると、サーバーが `utf8mb4_0900_ai_ci` を知らないので、デフォルトの `utf8mb4_general_ci` にfallbackする。

フォールバックが発生する組み合わせでは、 **collationどころかcharacter setすら一致しなくなります。**

## collation不整合への対処方法

collationに特にこだわりがなくて、とりあえずサーバーのデフォルトのcollationを使いたい場合は、 `SET NAMES utf8mb4` コマンドを実行するのが良いでしょう。

接続をコネクションプールやO/Rマッパーが管理している場合は、新規接続時に設定用のコマンドを発行する機能がないかを調べて見てください。

利用するcollationを指定したい場合は、 `CREATE DATABASE <データベース名> COLLATE <collation名>` のようにデータベースのデフォルトcollationを設定し、接続のcollationは `SET NAMES utf8mb4 COLLATE <collation名>` のように指定するといいでしょう。

### SET NAMES のセキュリティリスクについて

接続のcollationは基本的にサーバー側でしか利用されないのですが、charsetはSQL文字列のエスケープに利用されるので、クライアントとサーバーの両方で一致している必要があります。

MySQLのCライブラリの場合、 `mysql_real_escape_string()` という関数がクライアント側の接続charsetを利用するのですが、 `SET NAMES` を普通のクエリとして実行するとクライアント側の接続charsetが更新されないままサーバーのcharsetが変更されるので、SQLインジェクションのリスクがあります。

そこで、 `SET NAMES` クエリを実行する代わりに、 `mysql_set_character_set()` という関数を使うことが推奨されています。この関数は `SET NAMES` クエリを実行しつつ、クライアント側の接続charsetを更新します。

しかし、 `mysql_set_character_set()` では `SET NAMES utf8mb4 COLLATE <collation名>` クエリを実行できません。推奨に従いつつcollationを設定するには、 `mysql_set_character_set()` を実行した後に `SET NAMES ... COLLATE ...` クエリを実行する必要があります。（順番が逆になると、そのcharsetのデフォルトのcollationに変更されてしまいます。）

実はこのセキュリティリスクは、新規接続時のcharset指定にもあります。上で見たように、クライアントが utf8mb4 を利用しているつもりでも、サーバーが指定されたcollation_idを知らない（あるいは `skip-character-set-client-handshake` が設定されている）ためにデフォルトの接続charset/collationにフォールバックすることで、クライアントとサーバーでcharsetが一致しなくなります。

とはいえ、実際にエスケープ処理で問題が発生するのは、 "sjis" などのマルチバイト文字の一部のバイトがASCIIコードでエスケープ対象になるバイトと衝突するcharsetを利用する場合のみです。UTF-8では問題にならないので、あまり気にしなくても良いでしょう。

例えば、新規接続時に `utf8mb4_0900_ai_ci` を指定してサーバー側が `utf8mb3_general_ci` や `latin1_swedish_ci` にフォールバックした場合、クライアント側の接続charsetはutf8mb4です。この状態で接続直後に `SET NAMES utf8mb4 collate utf8mb4_bin` を実行してもエスケープの問題は発生しません。その後はサーバー側とクライアント側のcharsetが一致します。もし `SET NAMES` が失敗したら、新規接続のハンドシェイクの場合と異なりエラーになるので、charset不整合を見逃すことはありません。なので `mysql_set_character_set()` を使用しなくても安全ですし、ラウンドトリップを1回減らせます。

### 別の方法: `skip-character-set-client-handshake`

クライアントが多くて全てに対策するのが困難な場合は、サーバー側で対処することも可能です。

サーバー側のデフォルトのcharset/collationがアプリケーションで利用したいものになっていることを確認した上で、 `skip-character-set-client-handshake` を有効にして再起動すれば、ハンドシェイクを無視して強制的に全クライアントのcharset/collationを統一できます。

もちろん、 'sjis' で接続していて、しかも `mysql_set_character_set()` を使わずにハンドシェイクでcharsetを指定しているようなアプリケーションがある場合はこの設定は危険です。全部のクライアントがUTF-8を使っていて、collationやmb3/mb4まで統一したい場合にのみおすすめします。

## 対策の実例: mysqlclient

私がメンテしている、Python用のMySQL接続ライブラリである [mysqlclient](https://pypi.org/project/mysqlclient/)では、2021-11-18リリースのv2.1.0から新規接続時に自動で `mysql_set_character_set()` を呼び出すようにしました。

MySQL互換プロトコルを利用していて、 `SET NAMES` クエリに対応していないサーバーがあった場合はエラーになってしまうのですが、現在までにそういったエラー報告は受けていません。MySQLドライバーを開発されている方は参考にしてください。
