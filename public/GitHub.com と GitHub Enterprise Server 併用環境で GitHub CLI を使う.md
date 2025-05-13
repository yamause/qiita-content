---
title: GitHub.com と GitHub Enterprise Server 併用環境で GitHub CLI を使う
tags:
  - GitHub
  - GithubEnterprise
  - GithubCLI
private: false
updated_at: '2025-05-13T21:57:20+09:00'
id: b91308633404042b977c
organization_url_name: null
slide: false
ignorePublish: false
---

# 概要

GitHub では GitHub CLI という GitHub上のリポジトリをCLIで操作できるコマンドが提供されています。このコマンドは GitHub Enterprise版でも利用することが可能であり、セルフホスト型の GitHub Enterprise Server でも同様に動作します。

複数のプラットフォームを併用した環境で GitHub CLI を利用し、アカウントを切り替える際に発生する問題とその解決策について記載します。

# 問題

GitHub CLI は複数のアカウントでのログインをサポートしており、 `gh auth switch` コマンドでアカウントを切り替えることが可能です。このとき、GitHub.com または、GitHub Enterprise Server のどちらか一方のプラットフォームだけを利用している場合は特に問題はありません。しかし、複数のプラットフォームでアカウントを切り替えて利用しようとすると問題が発生します。

GitHub CLI に複数のプラットフォームのアカウントが設定されているとき、GitHub CLI はデフォルトで GitHub.com の情報を参照するようになっています。つまりこの状況の時、GitHub Enterprise Server 上のリポジトリを参照したいにもかかわらず、GitHub CLI は GitHub.com 上のリポジトリしか参照しない。ということです。

そのため、複数のプラットフォームを併用している場合、GitHub CLI のターゲットホストを切り替える必要があります。しかし、現状の GitHub CLI にはそのようにターゲットホストを切り替える機能を持ちません。

このように `gh auth status` でアカウント情報は確認できますが `gh auth switch` ではターゲットホストを切り替えることはできません。

```
$ gh auth status
github.com
  ✓ Logged in to github.com account example_user (/home/example_user/.config/gh/hosts.yml)
  - Active account: true
  - Git operations protocol: https
  - Token: gho_************************************
  - Token scopes: 'gist', 'read:org', 'repo', 'workflow'

git.example.com
  ✓ Logged in to git.example.com account example_user (/home/example_user/.config/gh/hosts.yml)
  - Active account: true
  - Git operations protocol: https
  - Token: gho_************************************
  - Token scopes: 'gist', 'read:org', 'repo', 'workflow'
```

この件に関してISSUEも上がっており、想定通りの動作であるとコメントされています。

https://github.com/cli/cli/issues/10058

> 原文
What you're describing is the expected behaviour, as far as I can tell from your explanation. Logging in to a host doesn't switch the targeted host, it only stores a token. If you have authenticated against more than one host, `gh` will default to targeting github.com.

> 日本語訳（Google翻訳）
ご説明いただいた内容は、ご説明から判断する限り、想定通りの動作です。ホストにログインしても、ターゲットホストは切り替わらず、トークンが保存されるだけです。複数のホストで認証している場合、gh はデフォルトで [github.com](http://github.com/) をターゲットとします。

ここまでの話をまとめます。

- GitHub CLI にはターゲットホストを切り替える機能がない
- GitHub.com と GitHub Enterprise Server を設定するとGitHub CLI はデフォルトで GitHub.com だけを参照するようになっている。


# 解決策

GitHub CLI は環境変数 `GH_HOST` が設定されている場合、そのエンドポイントを参照する仕様になっています。

そのため、環境変数を設定することでターゲットホストを切り替えることができます。

```
export GH_HOST="git.example.com"
```

より切り替えを簡単にするために、つぎのようにエイリアスを設定し、 `~/.bashrc` などに登録する方法もあります。こうすることで既存の環境を汚さずにエイリアスを用いた GitHub CLI の実行時にのみ変数をセットすることができます。

```
alilas ghe="GH_HOST=git.example.com gh"
```

# あとがき

最近、弊社でも AI コーディングの波を受け、GitHub.com を利用することが推奨されるようになりました。私の所属するチームでは主に GitHub Enterprise Server を利用していましたが、現在移行作業を進めています。そのため、二つのプラットフォームを並行して運用する中でアカウント切り替えでこの問題に遭遇したため、その際に調べたことを記事にまとめてみました。

# 参考情報

GitHub, GitHub Docs, GitHub プラットフォーム間での GitHub CLI の使用(2025-05-13)
https://docs.github.com/ja/enterprise-cloud@latest/github-cli/github-cli/using-multiple-accounts
