---
title: Ansible × cloud-init
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# cloud-init × Ansible - VM作成時にAnsibleを自動実行しよう

こちらはAnsible Advent Calendar 2025 14日目の記事です。

---

クラウドやハイパーバイザーなどから作成した仮想マシンの初期構成を自動化するためにcloud-initを利用することがよくあります。

実はこのcloud-initでは[バージョン22.3](https://github.com/canonical/cloud-init/releases/tag/22.3)からAnsibleモジュールが公式に提供されており、これを利用することでcloud-initの処理の中でAnsibleを呼び出すことができます。

cloud-initだけでも十分ですが、Ansibleを呼び出すことで次の利点があります。

- **高度な構成管理**: cloud-initだけを利用するよりもAnsibleの機能を使うことで複雑な構成を簡単に定義できる
- **再利用がしやすい**: 既存のPlaybookやロールを再利用できる
- **構成の一貫性**: cloud-initを利用できない環境のサーバーとも同じPlaybookを利用することで構成の一貫性を保てる

## Ansible Playbookの準備

今回の検証で利用するPlaybookを作成します。

サンプルコードは公開していますのでそのままご利用いただけます。
https://github.com/yamause/ansible-advent-calendar-2025/blob/main/playbooks/site.yml

このPlaybookでは次のことを行います。

- cowsayパッケージのインストール
- cowsayコマンドの実行
- `/tmp/advent_message.txt` に実行結果を保存

```yaml
# site.yml
---
- name: Ansible Advent Calendar 2025 Day 14 - cloud-init example playbook
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Install cowsay package
      ansible.builtin.package:
        name: cowsay
        state: present

    - name: Display the Advent Calendar message using cowsay
      ansible.builtin.command: /usr/games/cowsay -f tux "Happy Holidays from Ansible Advent Calendar 2025!"
      register: cowsay_output
      changed_when: cowsay_output.rc != 0

    - name: Create a Advent Calendar message file
      ansible.builtin.copy:
        dest: /tmp/advent_message.txt
        content: "{{ cowsay_output.stdout }}"
        mode: '0644'
```

## cloud-configを書いてみよう

これが今回のサンプルコンフィグの全体です。詳細を見ていきましょう。

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - git

ansible:
  install_method: pip
  package_name: ansible-core
  setup_controller:
    repositories:
      - source: https://github.com/yamause/ansible-advent-calendar-2025.git
        path: /root/ansible-advent-calendar-2025
    run_ansible:
      - playbook_name: site.yml
        playbook_dir: /root/ansible-advent-calendar-2025/playbooks
```

GitHubからAnsible Playbookをダウンロードしてくる必要があるため、事前にgitをインストールします。

```yaml
package_update: true
package_upgrade: true
packages:
  - git
```

`install_method` はAnsibleをインストールする方法を `distro` または `pip` から選択します。

distroの場合は、aptなどのパッケージ管理ツールでインストールします。

pipの場合は `run_user` を指定しているかどうかによって動作が変わります。指定されている場合は指定したユーザーのユーザー空間にインストールします。指定されていない場合はシステムPythonにインストールします。

```yaml
ansible:
  install_method: pip
  package_name: ansible-core
  # run_user: ansible-user
```

`setup_controller.repositories` はAnsible Playbookを格納しているリポジトリの情報を記述します。

`source` はリモートリポジトリのURLを指定します。下記サンプルコードのリポジトリは公開していますので、そのままご利用いただけます。
`path` はリポジトリを保存するディレクトリパスを指定します。既存のパスを選択するとエラーになってしまうため、まだ作成されていないディレクトリパスを指定してください。

```yaml
ansible:
  ...
  setup_controller:
    repositories:
      - source: https://github.com/yamause/ansible-advent-calendar-2025.git
        path: /root/ansible-advent-calendar-2025
```

`setup_controller.run_ansible` は実行するPlaybookと実行パスを指定します。

`playbook_name` 実行するPlaybookを指定します。
`playbook_dir` Playookが格納されているディレクトリパスを指定します。

```yaml
ansible:
  ...
  setup_controller:
    ...
    run_ansible:
      - playbook_name: site.yml
        playbook_dir: /root/ansible-advent-calendar-2025/playbooks
```

## 実際に動かしてみよう

今回はAWSのEC2を利用して実行してみましょう。

UbuntuのAMIを利用して仮想マシンを作成します。

ちなみにAmazon Linux 2023でも試してみたんですが、cloud-initのバージョンが22.2でまだAnsibleモジュールの実装のないバージョンでした。

**高度な詳細** のトグルを開き画面を下のほうにスクロールすると **ユーザーデータ - オプション** があります。個々のテキストボックスに先ほどのcloud-configを張り付けるかファイルからアップロードしてください。

あとは仮想マシンを起動したら自動的にcloud-initが実行されます。

![alt text](img/newArticle001-01.png)

## 実行結果を確認しよう

先ほど作成した仮想マシンにログインします。

cloud-init statusコマンドで現在の実行状況がわかります。 `status: done` となっていれば実行が完了しています。

```sh
$ cloud-init status
status: done
```

Playbookで定義したファイルが無事に作成されています！

```sh
$ cat /tmp/advent_message.txt
 ____________________________________
/ Happy Holidays from Ansible Advent \
\ Calendar 2025!                     /
 ------------------------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
```

## こんなときは？

ここからは運用するにあたってよく遭遇する疑問にお答えします。

### Ansible Galaxyからコレクションやロールをインストールしたい

標準外のコレクションを利用したい場合は、インストールする必要があります。
下記の例では、依存するコレクションとnewrelicのロールをAnsible Galaxyでインストールしています。

```yaml
ansible:
  ...
  galaxy:
    actions:
      - ['ansible-galaxy', 'collection', 'install', 'ansible.utils', 'ansible.windows']
      - ['ansible-galaxy', 'role', 'install', 'newrelic.newrelic_install']
```

### Ansibleの実行オプションを指定したい

ansible-playbookの実行オプションを指定するには `run_ansible` の下で指定します。

```yaml
ansible:
  ...
  setup_controller:
    ...
    run_ansible:
      - playbook_name: hoge.yml
        playbook_dir: /root/hoge/playbooks/
        vault_password_file: /root/.vault_pass.txt
        inventory: /root/hoge/inventory/
        limit: "hoge"
```

これはソースコードを確認するとわかるのですが、run_ansibleに与えられたキーと値をそのまま実行オプションとしてコマンドに渡しています。そのため、[cloud-init公式ドキュメント](https://cloudinit.readthedocs.io/en/latest/reference/modules.html#ansible)に記載されているオプション以外も指定できます。

```python
def ansible_controller(cfg: dict, ansible: AnsiblePull):
    for repository in cfg.get("repositories", []):
        ansible.do_as(
            ["git", "clone", repository["source"], repository["path"]]
        )
    for args in cfg.get("run_ansible", []):
        playbook_dir = args.pop("playbook_dir")
        playbook_name = args.pop("playbook_name")
        command = [
            "ansible-playbook",
            playbook_name,
            *[f"--{key}={value}" for key, value in filter_args(args).items()],
        ]
        ansible.do_as(command, cwd=playbook_dir)
```

> [GitHub. canonical/cloud-init. cc_ansible.py](https://github.com/canonical/cloud-init/blob/78c68f5934c84dfa0dcce2cd1e755d1a420f7431/cloudinit/config/cc_ansible.py#L319)

### GitHubリポジトリに認証が必要な場合

リポジトリを非公開（PrivateまたはInternal）にしている場合、リポジトリをダウンロードするのに認証が必要になります。

GitHubの場合、Personal Access Token（PAT）をURLに挿入することで認証を通すことができます。
リポジトリの読み取り権限があれば十分ですので、PATを発行する際は必要最小限の権限を設定してください。

ただし、この方法の場合だとcloud-init.logにURLと一緒にPATが出力されてしまいます。

```yaml
ansible:
  ...
  setup_controller:
    repositories:
      - source: https://x-access-token:<your_token>@github.com/yamause/ansible-advent-calendar-2025.git
        path: /root/ansible-advent-calendar-2025
```

もしログに認証情報を出力させたくない場合は、事前に認証情報をGitの設定に登録しておく方法があります。
ただし、こちらもcloud-configや.git-credentialsに平文で保存されることになるので取り扱いには十分に注意してください。

ほかにいい方法があれば教えてください！

```yaml
#cloud-config
write_files:
  # PlaybookをGitHubからダウンロードするためにgitの認証情報を保存したファイルを作成します。
  - path: /root/.git-credentials
    owner: root:root
    permissions: '0600'
    content: |
      https://x-access-token:<your_token>@github.com

  # Gitの認証情報ヘルパーを設定するためのファイルを作成します。
  - path: /root/.gitconfig
    content: |
      [credential]
              helper = store
    permissions: '0644'
    owner: root:root
```

## あとがき

私は自宅で運用しているProxmoxVEの構成管理をTerraformで行っています。それとあわせてcloud-initを利用してAnsibleを呼び出すことでVMの作成から初期設定までを完全に自動化できています。

terraform applyを実行するだけで、すべての設定が終わるためVMを構成するときの手順を意識する必要がなくとても快適です。この記事が皆様のインフラ自動化のヒントになれば幸いです。

## 参考

[GitHub. canonical/cloud-init](https://github.com/canonical/cloud-init)
[Cloud-init documentation](https://cloudinit.readthedocs.io/en/latest/index.html)
