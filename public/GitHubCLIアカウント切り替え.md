---
title: 【Git ＆ GitHub CLI】ディレクトリ毎にアカウントを切り替えたい！
tags:
  - 'Git'
  - 'GitHub'
  - 'GitHub CLI'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

業務利用のアカウントと個人利用のアカウントを使い分けていると誤ってプッシュしてしまう...なんて事故を起こしそうになることがたびたび...
安全に運用するために業務アカウントと個人アカウントを分けられる仕組みを作りましょう！

※セキュリティに十分に気を配り、組織の規則に従ったアカウント運用をしてください。

# 設定

今回設定するディレクトリツリーです。

```
.
├── .gitconfig
├── .gitconfig_work     # 業務用設定
├── .gitconfig_private  # 個人用設定
└── Dev/
    ├── work/           # 業務用ディレクトリ
    └── private/        # 個人用ディレクトリ
```

- `~/Dev/work` の配下で業務用のリポジトリを管理します。
- `~/Dev/private` の配下で個人用のリポジトリを管理します。

それぞれのディレクトリに移動したとき、自動的に Git & GitHub CLI アカウントを切り替えます。

## Gitアカウントを使い分ける

まずはGitアカウントの使い分けから

### includeIf 設定をする

includeIf は条件付きで別の構成ファイルを取り込むことができる機能です。[^1]これを利用して指定したディレクトリにいるときに異なる設定を使い分けます。

`~/.gitconfig` を設定します。

- `[includeIf "gitdir:<directory>"]` ：固有の設定を適用したいディレクトリのパスを指定します。
- `path = <config-file>` ：固有の設定を記述したファイルパスを指定します。

```sh
[includeIf "gitdir:~/Dev/work/"]
    path = ~/.gitconfig_work

[includeIf "gitdir:~/Dev/private/"]
    path = ~/.gitconfig_private
```

### アカウントを登録する

`~/.gitconfig_work` に業務用のアカウントを登録します。同様に `~/.gitconfig_private` に個人用のアカウントを登録します。

```ini:~/.gitconfig_work
[user]
    name = <業務アカウント>
    email = <業務メールアドレス>
```

```ini:~/.gitconfig_private
[user]
    name = <個人アカウント>
    email = <個人メールアドレス>
```

git init を実行することでこのディレクトリ配下のすべてのディレクトリに格納されたリポジトリで設定が反映されます。

```sh
cd ~/Dev/work
git init
```

それぞれのディレクトリに移動し、想定したアカウントになっているか確認してみましょう。

```sh
git config user.name
git config user.email
```

## GitHubアカウントを使い分ける

つづいてGitHubアカウントの使い分けを紹介します。

`gh auth login` で認証したアカウントよりも環境変数 `GH_TOKEN` に登録されているトークンを所有するアカウントが優先して適用されます。これを利用してアカウントを切り替えていきます。

ちなみに `GH_TOKEN` が登録されていると常にそのアカウントが優先されるようになるため `gh auth switch` コマンドでアカウントを切り替えることができなくなります。※該当のディレクトリにいるときにのみ環境変数を設定するため今回これはあまり問題になりません。

```sh
gh auth status
github.com
  # 環境変数から読み込んだアカウントが優先され Active になっている
  ✓ Logged in to github.com account yamause (GH_TOKEN)
  - Active account: true
  - Git operations protocol: ssh
  - Token: gho_************************************
  - Token scopes: 'admin:public_key', 'gist', 'read:org', 'repo', 'workflow'

  ✓ Logged in to github.com account yamause (/home/yamause/.config/gh/hosts.yml)
  - Active account: false
  - Git operations protocol: ssh
  - Token: gho_************************************
  - Token scopes: 'admin:public_key', 'gist', 'read:org', 'repo', 'workflow'

# ↑プライベートなPCからコマンド実行しているのでこの例で表示されているのは同じアカウントですが
#  異なるアカウントにログインしている場合も同様に GH_TOKEN に設定されているアカウントが優先されます。
```

### GitHub CLI ログイン

はじめに利用するGitHubアカウントにGitHub CLIからログインをしておきます。

```sh
gh auth login
```

### direnv の設定

direnv を利用してディレクトリに入ったときに自動的に環境変数 `GH_TOKEN` が設定されるようにします。

direnv とは現在のディレクトリまたは親ディレクトリに配置され direnv によって承認された `.envrc` ファイルに記述された環境変数を自動的に設定できる優れものです。[^2]

direnv のインストール方法は下記のドキュメントを参考にしてください。

https://direnv.net/docs/installation.html


それぞれのディレクトリに移動し、`direnv edit` コマンドを実行します。コマンドを実行するとエディタが開かれます。

```sh
cd ~/Dev/work
direnv edit .
```

`gh auth token --user <アカウント名>` は指定したログイン済みユーザーのトークンを出力するコマンドです。これを `GH_TOKEN` に設定します。

`~/Dev/work` の `.envrc` には、業務アカウントのトークンを設定します。

```sh:~/Dev/work/.envrc
export GH_TOKEN=$(gh auth token --user <業務アカウント名>)
```

同様に、`~/Dev/private` の `.envrc` には、個人アカウントのトークンを設定します。

```sh:~/Dev/private/.envrc
export GH_TOKEN=$(gh auth token --user <個人アカウント名>)
```



以上！

これでディレクトリに移動するだけでGitもGitHubも自動的にアカウントが切り替わります！

# あとがき

わたしの所属している組織は GitHub Enterprise Server から GitHub Enterprise Cloud への移行期でありふたつの環境が併用されています。今回の手法はそのような環境でもとても有効でした。（わたしの所属するチームは最近やっとリポジトリをすべて移行させることができました！）


# 関連記事

https://qiita.com/yamause/items/b91308633404042b977c

# 参考

[direnv - @zimbatm](https://direnv.net/)
[Git Documentation - Git and Software Freedom Conservancy](https://git-scm.com/doc)

[^1]: Git and Software Freedom Conservancy. “git-config” .Git - Documentation. 2025-08-18 . https://git-scm.com/docs/git-config#_conditional_includes, (閲覧 2025-09-10) .
[^2]: @zimbatm. “direnv – unclutter your .profile”. direnv. 2025-08-18 . https://direnv.net/, (閲覧 2025-09-10) .