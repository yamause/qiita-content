---
title: Ansible × cloud-init
tags:
  - 'Ansible'
  - 'cloud-init'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# cloud-init × Ansible - VM作成時にAnsibleを自動実行しよう

こちらはAnsible Advent Calendar 2025 14日目の記事です。

---

クラウドやハイパーバイザーで作成した仮想マシンの初期構成を自動化する際、cloud-initを利用するケースが多くあります。

cloud-initでは[バージョン22.3](https://github.com/canonical/cloud-init/releases/tag/22.3)からAnsibleモジュールが公式に提供されており、cloud-initの処理の中でAnsibleを実行できるようになりました。

cloud-initだけでも十分ですが、Ansibleを呼び出すことで次の利点があります。

- **高度な構成管理**: Ansibleの豊富な機能により、複雑な構成を簡潔に定義できる
- **再利用性**: 既存のPlaybookやロールをそのまま活用できる
- **構成の一貫性**: cloud-initを利用できない環境でも同じPlaybookを使用することで、構成管理を統一できる

## Ansible Playbookの準備

今回の検証で使用するPlaybookを作成します。

サンプルコードはGitHubで公開していますので、そのまま利用できます。
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
      changed_when: false

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

GitHubからAnsible Playbookをダウンロードするため、事前にgitをインストールします。

```yaml
package_update: true
package_upgrade: true
packages:
  - git
```

`install_method` はAnsibleをインストールする方法を `distro` または `pip` から選択します。

`distro`を選択すると、aptなどのパッケージ管理ツールを使ってインストールします。

`pip`を選択した場合、`run_user`の指定有無で動作が異なります。ユーザーを指定した場合はそのユーザーの環境にインストールし、指定しない場合はシステム全体のPython環境にインストールします。

```yaml
ansible:
  install_method: pip
  package_name: ansible-core
  # run_user: ansible-user
```

`setup_controller.repositories`では、Ansible Playbookを格納しているリポジトリの情報を記述します。`setup_controller`はPlaybookの実行環境をセットアップする設定で、GitリポジトリからPlaybookをダウンロードし、このホスト上で`ansible-playbook`コマンドを実行します。

- `source`: リモートリポジトリのURLを指定する。サンプルリポジトリは公開しているので、そのまま利用できる。
- `path`: リポジトリをクローンするディレクトリパスを指定する。既存のパスを指定するとエラーになるため、まだ存在しないパスを指定すること。

```yaml
ansible:
  ...
  setup_controller:
    repositories:
      - source: https://github.com/yamause/ansible-advent-calendar-2025.git
        path: /root/ansible-advent-calendar-2025
```

`setup_controller.run_ansible`では、実行するPlaybookと実行ディレクトリを指定します。

- `playbook_name`: 実行するPlaybookのファイル名を指定する。
- `playbook_dir`: Playbookが格納されているディレクトリパスを指定する。

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

なお、Amazon Linux 2023でも試しましたが、cloud-initのバージョンが22.2であり、Ansibleモジュールが実装されていませんでした。

**高度な詳細**のトグルを開き、画面下部の**ユーザーデータ - オプション**へ先ほどのcloud-configを貼り付けるか、ファイルからアップロードしてください。

仮想マシンを起動すると、自動的にcloud-initが実行されます。

![EC2インスタンス作成画面のユーザーデータ入力欄](img/newArticle001-01.png)

## 実行結果を確認しよう

先ほど作成した仮想マシンにログインします。

`cloud-init status`コマンドで現在の実行状況を確認できます。`status: done`と表示されていれば、実行が完了しています。

```sh
$ cloud-init status
status: done
```

Playbookで定義したファイルが無事に作成されています！

実行に失敗した場合は、以下のログファイルで詳細を確認できます。

```sh
# cloud-initの実行ログ
$ sudo cat /var/log/cloud-init.log

# cloud-initの出力ログ（Ansibleの実行結果も含む）
$ sudo cat /var/log/cloud-init-output.log
```

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

## 実践的な活用Tips

ここからは運用するにあたってよく遭遇する疑問にお答えします。

### Ansible Galaxyからコレクションやロールをインストールしたい

標準外のコレクションを利用する場合は、事前にインストールが必要です。
以下の例では、依存するコレクションとnewrelicのロールをAnsible Galaxyからインストールしています。

```yaml
ansible:
  ...
  galaxy:
    actions:
      - ['ansible-galaxy', 'collection', 'install', 'ansible.utils', 'ansible.windows']
      - ['ansible-galaxy', 'role', 'install', 'newrelic.newrelic_install']
```

### Ansibleの実行オプションを指定したい

`ansible-playbook`の実行オプションは、`run_ansible`配下で指定できます。

```yaml
ansible:
  ...
  setup_controller:
    ...
    run_ansible:
      - playbook_name: production.yml
        playbook_dir: /root/myapp/playbooks/
        vault_password_file: /root/.vault_pass.txt
        inventory: /root/myapp/inventory/
        limit: "webservers"
```

ソースコードを見ると、`run_ansible`に指定したキーと値がそのまま実行オプションとしてコマンドに渡されることがわかります。そのため、[cloud-init公式ドキュメント](https://cloudinit.readthedocs.io/en/latest/reference/modules.html#ansible)に記載されていないオプションも指定可能です。

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

リポジトリを非公開（PrivateまたはInternal）にしている場合、ダウンロード時に認証が必要です。

GitHubの場合、Personal Access Token（PAT）をURLに埋め込むことで認証できます。
リポジトリの読み取り権限があれば十分なので、PATを発行する際は必要最小限の権限を設定してください。

ただし、この方法ではcloud-init.logにURLとともにPATが記録されてしまう点に注意が必要です。

```yaml
ansible:
  ...
  setup_controller:
    repositories:
      - source: https://x-access-token:<your_token>@github.com/yamause/ansible-advent-calendar-2025.git
        path: /root/ansible-advent-calendar-2025
```

ログに認証情報を記録したくない場合は、事前にGitの設定に認証情報を登録する方法があります。
ただし、この方法でもcloud-configや.git-credentialsに平文で保存されるため、取り扱いには十分注意してください。

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

私は自宅で運用しているProxmox VEの構成管理をTerraformで行っています。それとあわせてcloud-initからAnsibleを呼び出すことで、VMの作成から初期設定までを完全に自動化しています。

`terraform apply`を実行するだけですべての設定が完了するため、構築手順を意識する必要がなく非常に快適です。この記事が皆様のインフラ自動化のヒントになれば幸いです。

## 参考

[GitHub. canonical/cloud-init](https://github.com/canonical/cloud-init)
[Cloud-init documentation](https://cloudinit.readthedocs.io/en/latest/index.html)
