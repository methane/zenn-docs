---
title: "PEP 582 (__pypackages__) がRejectされました"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python"]
published: false
---

`__pypackages__` を自動的にimport pathに追加する提案 PEP 582 が Reject されました。ただし "at least in its current form." という但し書きがついているので、今後形を変えて復活する可能性は残っています。

https://discuss.python.org/t/pep-582-python-local-packages-directory/963/430


## PEP 582 とは

https://peps.python.org/pep-0582/

提案されていた仕様を簡単に説明すると、次のようになります。

* `python` でインタラクティブシェルを実行した時はカレントディレクトリ、 `python path/to/script.py` を実行したとき (shebang経由で起動された場合を含む) は `script.py` と同じディレクトリに、 `__pypackages__` ディレクトリがあった場合、 import path に追加する。

このPEPのモチベーションは、仮想環境によってPythonの学習ハードルが高くなっている現状の解決です。外部のライブラリを利用し出すとすぐに仮想環境が必要になってきますが、仮想環境を使うにはシェルの知識が必要で、さらに activate 忘れによるトラブルに何度も直面してしまいます。


## PEP 582 は何を解決する/しないのか

残念ながら、現在のPythonのパッケージ管理のユーザー体験は決して褒められる物ではありません。

PEP 582 は、まるで `node script.js` が自動で `node_modules` を探すのと同じように `python script.py` が `__pypackages__` を探すので、Pythonのパッケージ管理体験が node.js に近づくような幻想を持たせてくれます。

しかし、実際には PEP 582 が採用されたとしても、Pythonのパッケージ管理が npm や Bundler に近づくとは言い難いのです。 `__pypackages__` が採用された場合、現状と比べておこる変化の予想を書きます。

### pip を使う場合の手間は変わるか？

venv の activate は不要になります。自動で activate するための `direnv` のようなツールを覚える必要はなくなります。

一方で、venvと違って `__pypackages__` は特定の Python バージョンとリンクされていないので、そのプロジェクトがどの Python バージョンを使っていたのかを記憶し、毎回その Python を使う必要があります。

それは現実的ではないので、実際には `pyenv local` のようなツールに頼ることになりますが、 `direnv` の代わりに `pyenv local` になるだけで、ハードルが下がったとは言えません。

### venv は不要にならない

PEP 582 は次のように述べています。

> This proposal should be seen as independent of virtual environments, not competing with them. At best, some use cases currently only served by virtual environments can also be served (possibly better) by __pypackages__.

`__pypackages__` は venv のユースケースの一部しかカバーしないので、引き続き venv は必要になります。Pythonユーザーはどこまで `__pypackages__` で可能で、どこから venv が必要になるのかを学び、選択しないといけなくなります。

また、Python関連の多くのツールも、 venv と別に `__pypackages__` への対応が必要になるでしょう。エコシステム全体の複雑さは増えることになります。

現状の PEP 582 が venv に比べて不足している主な点は以下の通りです。

* 前述した通り、特定のPythonインタープリタとリンクされないので、インタープリタの選択が別に必要になる。
* スクリプトと同じディレクトリからしか `__pypackages__` ディレクトリを探さないので、プロジェクトのサブディレクトリにスクリプトを置けなくなる。
* `bin/` に該当するディレクトリが決まってないので、インストールされたパッケージが提供しているスクリプトやコマンドを実行できない。 (`python -m module` は可能だが、 Ruff のようなPythonモジュールでないコマンドは無理)


### パッケージ管理ツールの乱立は変わらない

PEP 582 は、 pdm, flit, poetry, Hatch, pipenv などのツールが乱立している現状には何も影響を与えません。

ただし、これらのツールが PEP 582 に対応しているかどうかの違いが発生したり、これらのツールに PEP 582 を使うオプションが追加されたりという形で、全体的に複雑さが増える方向の変化は発生するでしょう。

ここまで見てきたように、現状の PEP 582 は便利になる場面が限定される一方で、エコシステム全体をより複雑な方に導くので、 Reject されたのは正しい判断だと思います。


## Python のパッケージ管理を良くするためには

コマンドラインシェルからPythonを利用する場合、Poetryのようなパッケージ管理ツールを使えば npm や Bundler と比べて面倒なことはありません。

問題は、パッケージ管理ツールが乱立していて、公式に推奨されたツールが無い事です。 Poetry が一番人気ではあるものの、 `pyproject.toml` にプロジェクト・メタデータを書く公式の方法 (PEP 621)　が決まる前に生まれたためにPEP 621に準拠していないなど、どれも一長一短なのです。

ユーザーが「一つの推奨されたパッケージ管理ツール」を求めていることは理解されているものの、残念ながら今の PyPA にはそれを決められるような体制にはなっていないのが現状です。

https://discuss.python.org/t/pypa-mechanism-for-making-official-recommendations/24744

一方で、 VSCode 等、コマンドラインシェル以外の場所からPythonを使う場合は、「このプロジェクトで使うデフォルトのvenv」がツール間で共有できる仕組みがあれば連携がスムーズになります。

その点に関連するのが PEP 704 です。このPEPは、パッケージインストーラー（pipなど）が、venvをactivateしていなくても自動で `.venv` を使うようにしようという提案です。

https://peps.python.org/pep-0704/

Pythonの外で開発されているツールのUXをPEPが決めるのはどうなんだとか、 conda は仮想環境に含まれるかとかいった問題があって PEP 704 自体が進む様子はありません。しかし、これを機に `.venv` というディレクトリを公式推奨にしようという話は進んでいます。

https://discuss.python.org/t/pep-704-require-virtual-environments-by-default-for-package-installers/22846/177

