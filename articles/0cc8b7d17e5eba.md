---
title: "PythonのデフォルトエンコーディングをUTF-8にするために"
emoji: "🐍"
type: "tech"
topics:
  - "python"
  - "windows"
published: true
published_at: "2021-02-08 15:00"
---

Python がテキストファイルを開く時のデフォルトエンコーディングがUTF-8でないことは、多くのWindowsユーザー、特にプログラミング初心者にとって障害になっています。

UnicodeDecodeError で検索すると、多くのWindowsユーザーが問題に遭遇しているのがわかります。

* https://qiita.com/Yuu94/items/9ffdfcb2c26d6b33792e
* https://www.mikan-partners.com/archives/3212
* https://teratail.com/questions/268749
* https://github.com/neovim/pynvim/issues/443
* https://www.coder.work/article/1284080
* https://teratail.com/questions/271375
* https://qiita.com/shiroutosan/items/51358b24b0c3defc0f58
* https://github.com/jsvine/pdfplumber/issues/304
* https://ja.stackoverflow.com/questions/69281/73612
* https://trend-tracer.com/pip-error/

エラーの内容を見てみると、次のようなケースが多いようです。

* JSONなどのUTF-8で書かれたテキストファイルを開こうとして UnicodeDecodeError が発生する
* Webから取得したテキストデータなどを保存しようとして UnicodeEncodeError が発生する
* `pip install` しようとしたパッケージが `setup.py` の中で、UTF-8で書かれた README.md や LICENSE ファイルを読んでいる

これらのエラーは、Pythonがデフォルトで利用するエンコーディングをUTF-8にすると解決します。

しかし、これは後方互換性を失う変更になります。後方互換性のためのオプションを残しつつ変更しようという提案をしたことがあるのですが、Microsoft所属のコミッターであるSteve Dower氏から[強い反対](https://discuss.python.org/t/3122/16)を受けました。数年間、 `encoding=` を指定していない全ての `open()` などの呼び出しに、デフォルトが変わるというWarningを発生させるべきだと言うのです。

Warningに対する反対意見もあります。ほとんどの `encoding=` 指定がない `open()` は、ASCIIのみのファイルを開いていたり、クラスプラットフォームの必要がないケースだったりです。DeprecationWarningならほとんどの場合デフォルトで表示されないのですが、それでもDeprecationWarningが大量発生すると他のDeprecationWarningが巻き添えで無視されるケースが出てくるので良くありません。

Warningを出すべきという意見と嫌だと言う両方の意見の板挟みになり、このままでは前に進めません。どうしたら良いでしょうか？


## 戦略1: opt-in な Warning

大きな問題は、DeprecationWarningを出すのがどれくらいウザいのかです。そこで opt-in でEncodingWarningを出す提案[PEP 597](https://www.python.org/dev/peps/pep-0597/)を出しています。だいぶ話がまとまってきたので早ければ今週中にもSteering Councilに提出しようと思っています。

このオプションを有効にして、標準ライブラリやpipのような誰もが使うツールから不適切な `encoding=` の省略を潰していけば、いつかはこのWarningをデフォルトで出せるようになるかもしれません。うまくいけばその数年後にデフォルトのエンコーディングを変更できるでしょう。

しかし、この戦略には大きな欠点があります。デフォルトのエンコーディングを変更できるのがいつになるかわかりません。少なくとも5年以上は先になるでしょう。2030年に間に合わないかもしれません。

Windows上でPythonを使う多くの新しいユーザーを、５年以上もこの不幸な状況に留めておくのは良くありません。もっと素早い解決方法が必要です。


## 戦略2: UTF-8 mode を普及させる

UTF-8 mode を使うと最初に紹介した問題を解決することができます。ですが UTF-8 mode はまだあまり知られていません。

@[tweet](https://twitter.com/methane/status/1354612253505413120)

UTF-8 modeを有効にするには `-Xutf8` オプションか `PYTHONUTF8` 環境変数を利用する必要があり、これは特にコマンドラインを普段使わないようなユーザーにとっては高いハードルになります。インストーラーやスタートメニューに登録される小さいツールなどで設定できるようユーザー環境変数を設定することができますが、それでも既存の Python を使ったアプリを壊してしまう懸念があり、そう簡単に誰にでもおすすめできるものではありません。

この問題を解決するために、環境（インストールや仮想環境）ごとにUTF-8 modeを有効にできる設定ファイルを追加しようというアイデアを現在 Python-ideas MLで提案しています。

まだ具体的なアイデアは固まっていませんが、例えば `python.exe` と同じディレクトリに `python.ini` ファイルを置き、その中に `utf8mode=1` という行があればUTF-8 modeを有効にするというものです。

Pythonをインストールするときや仮想環境を作る時にUTF-8 modeを有効にするか選べるようにすれば、例えば授業で生徒にPython環境をセットアップしてもらうときにUTF-8 modeを有効にしてもらうことができるでしょう。

もう少し野心的な事を話すと、UTF-8 modeを有効にすることがWindows上でのPython環境のベストプラクティスと認識されるようになったら、インストーラーでこのオプションにデフォルトでチェックを入れることができるかもしれません。そうなれば、新しいユーザーにとっては実質的にUTF-8 modeがデフォルトになります。

初心者向けに Python の環境構築記事を書かれる方は、ぜひ UTF-8 mode を紹介する事を検討してみてください。