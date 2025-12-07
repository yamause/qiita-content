---
title: pip をプライベートな PyPIサーバーからインストールする方法（pypiserver）
tags:
  - Python
  - pip
  - package
private: false
updated_at: '2025-10-24T23:43:20+09:00'
id: e5586f17a20bed6868bf
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

自作したライブラリをローカル環境のPyPIサーバーからインストールしたい！と思いたったのでpypiserverで構築してみました。
基本的には公式のドキュメント通りにやれば問題ないですが何かの参考になれば幸いです。

つぎの手順で説明します。

1. pypiserverの構築
2. 自作パッケージの作成とパッケージのビルド
3. pypiserverへのパッケージのアップロード

# pypiserver の構築

pypiserverはPythonパッケージとしてPyPIに公開されており、pipコマンド経由でインストールできます。
そのほか、Dockerコンテナも公式にサポートされており、docker pullコマンドでダウンロードできます。
今回は、Dockerコンテナを用いてpypiserverを構築します。

https://hub.docker.com/r/pypiserver/pypiserver/tags/


pyserverのコンテナは、デフォルトのTCP/8080ポートで起動します。今回は `-p 8080:8080` オプションを使い、ローカルポートの8080番をここに割り当てます。
パッケージを保存するディレクトリをローカルのディレクトリでバインドマウントする場合は `-v ~/packages:/data/packages` オプションを追加します。
検証を簡単にするため .htaccessによる認証をスキップします。 `-a . -P .` オプションをつけることで認証を無効にしています。

```sh
docker run -p 8080:8080 -v ~/packages:/data/packages pypiserver/pypiserver:latest run -a . -P . /data/packages
```

コンテナが起動したら http://localhost:8080 にブラウザから接続します。
成功していればつぎのように表示されます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/634462/ba94003c-dc32-418d-bbdf-349c9d14305c.png)


# 自作パッケージを作成する

pypiserverにアップロードするための自作パッケージを作成します。
今回は、pypiserverの動作を確認することが目的であるため、ここではテスト用の単純なパッケージを作成します。

## パッケージとして配布するサンプルコードの作成

今回、CLIとして実行可能なスクリプトを作成します。


実行イメージ。

```sh
$ sample-project
hello world!!
```

パッケージのレイアウトはsrcレイアウトを採用しつぎのように作成します。

```tree
.
├── pyproject.toml
└── src
    └── sample_project
        ├── __init__.py  # ファイルだけ作成して処理は記述しません
        ├── __main__.py  # CLIで呼び出されるエントリーポイントを定義します。
        └── main.py  # メインの処理を記述します。
```

メインの処理を記載します。今回はパッケージングすることが目的なので中身は適当です。

```python
# ./src/main.py
def say(word = "world"):
    """SAMPLE Package

    Args:
        word (str, optional): say hello. Defaults to "world".
    """
    print(f"hello {word}!!")

if __name__ == "__main__":
    say()
```

CLIで呼び出されるエントリーポイントを定義します。

```python
# ./src/__main__.py
from . import main

main.say()
```

## pyproject.toml の作成

Pythonをパッケージングするための設定ファイルを記述します。

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "sample-project"
version = "0.0.1"
requires-python = ">=3.8"
description = "PyPI Server test"

[project.scripts]
sample-project = "sample_project.main:say"
```

## パッケージの確認

実際に動作するか確認してみましょう。
つぎのコマンドでローカルのファイルをインストールできます。
※必要に応じてvenvなど仮想環境を利用してください。

```sh
pip install -e .
```

これで実行できるようになっているはずです。試しに実行してみましょう。

```sh
$ sample-project
hello world!!
```

あとで、pypiserver経由でインストールを確認するため、ここでの確認が済んだらアンインストールしておきます。

```sh
pip uninstall sample-project
```

# pypiserver へパッケージをアップロードする

pypiserverへパッケージをアップロードするにはいくつかの選択肢があります。

 - SCPなどを利用しpypiserverに直接ファイルを配置する
 - setuptoolsを利用する
 - Twineを利用する
 - pypi-uploaderを利用する

今回はTwineを利用してアップロードする手順を記載します。

## Twine とは？

PyPI (Pythonのパッケージリポジトリ) にPythonパッケージを公開するためのツールです。
ビルドシステムに依存せず、ソースコードとバイナリ配布物をアップロードできます。

詳細についてはつぎの資料をご確認ください。

https://twine.readthedocs.io/en/stable/


## パッケージをビルドする

パッケージをビルドするためのツールをインストールします。

```sh
pip install build
```

インストールしたbuildを利用し、作成したパッケージをビルドします。

```sh
python -m build .
```

ビルドが完了すると `dist` ディレクトリ配下につぎのようなファイルが作成されます。

```tree
./dist
├── sample-project-0.0.1-py3-none-any.whl
└── sample-project-0.0.1.tar.gz
```

## アップロード

twineをインストールします。

```sh
pip install twine
```

パッケージをアップロードします。
今回は、ローカルのPyPIサーバーへアップロードするため、 `--repository-url` オプションでローカルのPyPIサーバーのURLを指定します。
最後の引数には前の手順でビルドしたパッケージファイルを指定します。コマンド実行時にユーザー名とパスワードの入力を促されますが、.htaccessを設定していないため、そのままEnterでOKです。

```sh
twine upload --repository-url https://localhost:8080 dist/*
```

http://localhost:8080/packages/ に接続するとアップロードしたパッケージを確認できます。

# ローカル pypiserver からpipインストールする

すべての準備が整いました！ついにpipインストールを実行するときです！
つぎのコマンドを実行します。

```sh
pip install --extra-index-url http://localhost:8080 sample-project
```

ちなみにつぎの環境変数を設定するか、` ~/.pip/pip.conf` に設定を書き込むことで、`--extra-index-url` オプションを省略できます。


```sh
export PIP_EXTRA_INDEX_URL=http://localhost:8080/
```

```ini
[global]
extra-index-url = http://localhost:8080/
```

インストールできたら最後の確認です。

```sh
$ pip list
Package        Version
-------------- -------
pip            25.1.1
sample-project 0.0.1
```

無事にインストールしたパッケージのコマンドを実行できました！

```sh
$ sample-project
hello world!!
```

# あとがき

自分は基本的にGitHubのプライベートリポジトリから直接pipで引っ張ってくることが多いので正直そこまで必要ではないんですが、
プライベートのpipリポジトリにはほかにもdevpiやAWS CodeArtifactなどいろいろと選択肢があるようです。
なんならただの静的サイトにパッケージ配置するだけでもよかったりするのでお好みに合わせて利用してください。

# 参考

pypiserver PyPI.
https://pypi.org/project/pypiserver/

PyPA. Python Packaging User Guide.
https://packaging.python.org/en/latest/specifications/pypirc/
