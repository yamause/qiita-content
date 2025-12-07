---
title: AnsibleでAIを利用するモジュールを自作する
tags:
  - Python
  - Ansible
  - AI
  - Gemini
  - 生成AI
private: false
updated_at: '2025-10-24T23:43:21+09:00'
id: 0b92eb8417a4d7746698
organization_url_name: null
slide: false
ignorePublish: false
---

# 概要

AnsibleのPlaybookからGeminiなどのAIアシスタントへ問い合わせをするモジュールを自作します。

Ansibleモジュールとは普段Playbookにタスクとして記述している処理のまとまりのことです。モジュールはAnsibleによってターゲットノードに送信されターゲットノード上で実行されます。[^Introduction-to-modules]

```yaml
- name: Example from an Ansible Playbook
  ansible.builtin.ping:  # これがAnsibleモジュール
```

# ゴール

```yaml
- name: Hello AI
  ai:
    text: "こんにちは！Ansibleからの挨拶です。"
    api_key: "**********************"
    model: "gemini-2.5-flash"
  register: result

- name: Print the result
  ansible.builtin.debug:
    msg: "{{ result.message }}"
  when: result.message is defined
```

このようなPlaybookを実行すると、以下の結果になります。

```text
PLAY [Test Playbook] ***********************************************************************************************

TASK [Hello AI] ****************************************************************************************************
ok: [localhost]

TASK [Print the result] ********************************************************************************************
ok: [localhost] => {
    "msg": "こんにちは！Ansibleからのご挨拶、ありがとうございます！\n\n自動化と構成管理の強力な味方ですね。いつもお世話になっております！\n\n何かお手伝いできることはありますか？どのようなご用件でしょうか？"
}

PLAY RECAP *********************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

# 実装

Gemini APIを利用します。前提として、Googleアカウントを保有しており、APIキーの発行ができる状態でいる必要があります。

APIキーの発行は [Google AI Studio](https://aistudio.google.com/app/apikey) から行ってください。

## ディレクトリ構成

ディレクトリ構成は以下の通りです。

```text
project
├── library
│   └── ai.py  # 自作するAnsibleモジュール
└── site.yml  # 作成したモジュールを実行するための Playbook
```

- **library/ai.py**: 自作するAnsibleモジュールである。今回は説明を簡単にするためコレクションではなくスタンドアロンのローカルモジュールとして実装する。Playbookと同じディレクトリ内の `library` の下に配置する。[^Adding-modules-and-plugins-locally][^default-module-path]
- **site.yml**: 作成したモジュールを実行するためのPlaybookである。

## Ansibleモジュールの作成

1行目にshebang `#!/usr/bin/python` を記述します。これは `ansible_python_interpreter` が機能するために必要です。[^python-shebang-utf-8-coding]

つづいて利用するパッケージをインポートします。今回は追加のインストールが不要な標準パッケージだけを利用しています。これは、このモジュールを利用したPlaybookを実行する際に、Pythonの依存関係による問題を避けるためです。

```python:library/ai.py
#!/usr/bin/python

import json
import urllib.error
import urllib.request
from abc import ABC, abstractmethod

from ansible.module_utils.basic import AnsibleModule, env_fallback
```

### AIアシスタントへのリクエスト

将来的に複数のAIアシスタントに対応させられるように抽象クラス `AiAgent` を作成します。
共通の関数 `process_text` を定義します。受け取った引数の文字列をAIアシスタントに問い合わせ、応答を文字列として返すシンプルなものです。

```python:library/ai.py:AiAgent
class AiAgent(ABC):
    @abstractmethod
    def process_text(self, text: str) -> str:
        pass
```

Geminiを利用する具象クラス `GeminiAgent` を作成します。

処理中にエラーが発生する場合は `fail_json()` 関数を利用します。これにより、Ansibleに問題が発生したことを伝えることができます。それ以外に特別なことはしていません。Gemini APIの仕様にそってHTTP POSTリクエストを送信するだけです。

```python:library/ai.py:GeminiAgent
class GeminiAgent(AiAgent):
    def __init__(self, module: AnsibleModule, api_key: str, model="gemini-2.5-flash"):
        self.module = module
        self.api_key = api_key
        self.model = model
        self.base_url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent"

    def process_text(self, text: str) -> str:
        headers = {
            "x-goog-api-key": self.api_key,
            "Content-Type": "application/json"
        }
        payload = {
            "contents": [
                {"parts": [{"text": text}]}
            ]
        }
        data = json.dumps(payload).encode('utf-8')

        request = urllib.request.Request(
            self.base_url, data=data, headers=headers, method="POST")

        try:
            # リクエストを送信
            with urllib.request.urlopen(request) as response:
                # レスポンスデータを読み込み、JSONとしてパース
                response_body = response.read().decode('utf-8')
                response_json = json.loads(response_body)

                # レスポンスから生成されたテキストを抽出
                if 'candidates' in response_json and len(response_json['candidates']) > 0:
                    generated_text = response_json['candidates'][0]['content']['parts'][0]['text']
                    return generated_text
                else:
                    self.module.fail_json(
                        msg="Gemini API response does not contain candidates.")

        except urllib.error.HTTPError as e:
            # HTTPエラー（4xx, 5xxなど）の処理
            self.module.fail_json(
                msg=f"Gemini API returned HTTP error: {e.code} - {e.reason}")

        except urllib.error.URLError as e:
            # その他のURL関連エラーの処理
            self.module.fail_json(msg=f"Gemini API URL error: {e.reason}")

        except Exception as e:
            # その他の一般的なエラーの処理
            self.module.fail_json(
                msg=f"An unexpected error occurred: {str(e)}")
```

### Ansibleモジュールの定義

メイン処理を記述します。

<details><summary>python:library/ai.py:main 全体</summary>

```python:library/ai.py:main
def main():
    # モジュールの引数定義

    module_args = {
        "text": {
            "type": "str",
            "required": True
        },
        "api_key": {
            "type": "str",
            "required": False,
            "no_log": True,
            "fallback": (env_fallback, ['AI_API_KEY'])
        },
        "model": {
            "type": "str",
            "required": False,
            "default": "gemini-2.5-flash",
            "fallback": (env_fallback, ['AI_MODEL'])
        }
    }

    # 結果を格納する辞書
    result = {
        "changed": False,
        "message": "",
    }

    # AnsibleModuleオブジェクトの作成
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=False
    )

    # 引数から値を取得
    text: str = module.params['text']
    api_key: str = module.params['api_key']
    model: str = module.params['model']

    # AIエージェントのインスタンスを作成
    if model.startswith("gemini-"):
        agent = GeminiAgent(module, api_key, model)
    else:
        module.fail_json(msg=f"Unsupported model: {model}")

    # テキストを処理
    generated_text = agent.process_text(text)

    if generated_text is not None:
        result['message'] = generated_text
    else:
        module.fail_json(
            msg="Failed to process text with AI service.")

    # 結果を返す
    module.exit_json(**result)
```

</details>

`module_args` 変数にモジュールが受け取ることのできるパラメーターを辞書形式で定義します。パラメーター `api_key` と `model` に `fallback` オプションを利用しています。これにより、モジュールでパラメーターが指定されていないとき、環境変数が設定されていればその値を利用できます。

```python:library/ai.py:main
def main():
    # モジュールの引数定義

    module_args = {
        "text": {
            "type": "str",
            "required": True
        },
        "api_key": {
            "type": "str",
            "required": False,
            "no_log": True,
            "fallback": (env_fallback, ['AI_API_KEY'])
        },
        "model": {
            "type": "str",
            "required": False,
            "default": "gemini-2.5-flash",
            "fallback": (env_fallback, ['AI_MODEL'])
        }
    }
    ...
```

AnsibleModuleのコンストラクタの引数 `argument_spec` に `module_args` 変数を渡します。`supports_check_mode` は、Ansibleで `--check` などのオプションでのDry run実行をサポートするかを指定します。今回は `False` にしておきます。

これでモジュールのパラメーターを定義できました。

```python:library/ai.py:main
def main():
    ...
    # AnsibleModuleオブジェクトの作成
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=False
    )
    ...
```

`result` 変数に最終的にAnsibleに返す結果を辞書形式で定義します。ここで定義された値はAnsibleの `register` オプションなどで参照できます。 `changed` は予約されたパラメーターでシステムに変更を加えたかを表します。これは冪等性にかかわる重要な項目です。今回作成するモジュールはシステムへ変更を加えるものではないため `False` にしておきます。

```python:library/ai.py:main
def main():
    ...

    # 結果を格納する辞書
    result = {
        "changed": False,
        "message": "",
    }
    ...
```

ここからはAIアシスタントへリクエストを実行し、結果をAnsibleに返すまでを説明します。

Playbook内で指定されたモジュールのパラメーターを `module.params['param_name']` で取得します。

`model` が `gemini-` から始まる場合、前の手順で定義した `GeminiAgent` からインスタンスを作成します。それ以外の場合は、サポート外としてエラーを返すようにします。

`agent.process_text(text)` でリクエストを実行、 戻り値を `result['message'] ` に格納します。

結果を `exit_json()` 関数の引数に渡して実行します。これで結果がAnsibleに返されます。


```python:library/ai.py:main
def main():
    ...
    # 引数から値を取得
    text: str = module.params['text']
    api_key: str = module.params['api_key']
    model: str = module.params['model']

    # AIエージェントのインスタンスを作成
    if model.startswith("gemini-"):
        agent = GeminiAgent(module, api_key, model)
    else:
        module.fail_json(msg=f"Unsupported model: {model}")

    # テキストを処理
    generated_text = agent.process_text(text)

    if generated_text is not None:
        result['message'] = generated_text
    else:
        module.fail_json(
            msg="Failed to process text with AI service.")

    # 結果を返す
    module.exit_json(**result)
```

最後にスクリプト実行された場合のエントリーポイントを定義して完了です。

```python:library/ai.py
if __name__ == '__main__':
    main()
```

### library/ai.py 全体

<details><summary>python:library/ai.py 全体</summary>

```python:library/ai.py
#!/usr/bin/python

import json
import urllib.error
import urllib.request
from abc import ABC, abstractmethod

from ansible.module_utils.basic import AnsibleModule, env_fallback


class AiAgent(ABC):
    @abstractmethod
    def process_text(self, text: str) -> str:
        pass


class GeminiAgent(AiAgent):
    def __init__(self, module: AnsibleModule, api_key: str, model="gemini-2.5-flash"):
        self.module = module
        self.api_key = api_key
        self.model = model
        self.base_url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent"

    def process_text(self, text: str) -> str:
        headers = {
            "x-goog-api-key": self.api_key,
            "Content-Type": "application/json"
        }
        payload = {
            "contents": [
                {"parts": [{"text": text}]}
            ]
        }
        data = json.dumps(payload).encode('utf-8')

        request = urllib.request.Request(
            self.base_url, data=data, headers=headers, method="POST")

        try:
            # リクエストを送信
            with urllib.request.urlopen(request) as response:
                # レスポンスデータを読み込み、JSONとしてパース
                response_body = response.read().decode('utf-8')
                response_json = json.loads(response_body)

                # レスポンスから生成されたテキストを抽出
                if 'candidates' in response_json and len(response_json['candidates']) > 0:
                    generated_text = response_json['candidates'][0]['content']['parts'][0]['text']
                    return generated_text
                else:
                    self.module.fail_json(
                        msg="Gemini API response does not contain candidates.")

        except urllib.error.HTTPError as e:
            # HTTPエラー（4xx, 5xxなど）の処理
            self.module.fail_json(
                msg=f"Gemini API returned HTTP error: {e.code} - {e.reason}")

        except urllib.error.URLError as e:
            # その他のURL関連エラーの処理
            self.module.fail_json(msg=f"Gemini API URL error: {e.reason}")

        except Exception as e:
            # その他の一般的なエラーの処理
            self.module.fail_json(
                msg=f"An unexpected error occurred: {str(e)}")


def main():
    # モジュールの引数定義

    module_args = {
        "text": {
            "type": "str",
            "required": True
        },
        "api_key": {
            "type": "str",
            "required": False,
            "no_log": True,
            "fallback": (env_fallback, ['AI_API_KEY'])
        },
        "model": {
            "type": "str",
            "required": False,
            "default": "gemini-2.5-flash",
            "fallback": (env_fallback, ['AI_MODEL'])
        }
    }

    # 結果を格納する辞書
    result = {
        "changed": False,
        "message": "",
    }

    # AnsibleModuleオブジェクトの作成
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=False
    )

    # 引数から値を取得
    text: str = module.params['text']
    api_key: str = module.params['api_key']
    model: str = module.params['model']

    # AIエージェントのインスタンスを作成
    if model.startswith("gemini-"):
        agent = GeminiAgent(module, api_key, model)
    else:
        module.fail_json(msg=f"Unsupported model: {model}")

    # テキストを処理
    generated_text = agent.process_text(text)

    if generated_text is not None:
        result['message'] = generated_text
    else:
        module.fail_json(
            msg="Failed to process text with AI service.")

    # 結果を返す
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

</details>

## Playbook の作成

AIに挨拶をする簡単なPlaybookを作成します。

```yaml:site.yml
---
- name: Test Playbook
  hosts: localhost
  gather_facts: false
  tasks:

    - name: Hello AI
      ai:
        text: "こんにちは！Ansibleからの挨拶です。"
        # api_key: APIキーを指定または、環境変数 AI_API_KEY を設定
        # model: モデルを指定または、環境変数 AI_MODEL を設定 (デフォルト: gemini-2.5-flash)
      register: result

    - name: Print the result
      ansible.builtin.debug:
        msg: "{{ result.message }}"
      when: result.message is defined
```


# 実行してみる

事前にGemini APIキーを環境変数 `AI_API_KEY` に設定するか、Playbookで直接指定してください。

```shell
ansible-playbook project/site.yml
```

挨拶にお返事をもらうことができました！

```text
PLAY [Test Playbook] ***********************************************************************************************

TASK [Hello AI] ****************************************************************************************************
ok: [localhost]

TASK [Print the result] ********************************************************************************************
ok: [localhost] => {
    "msg": "こんにちは！Ansibleからのご挨拶、ありがとうございます！\n\n自動化と構成管理の強力な味方ですね。いつもお世話になっております！\n\n何かお手伝いできることはありますか？どのようなご用件でしょうか？"
}

PLAY RECAP *********************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```


## ちょっとした応用

`/proc/meminfo` の中身をAIアシスタントに確認してもらい、Markdown形式のレポートを作ってもらいます。

```yaml:site.yml
---
- name: Test Playbook
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Get Memory
      ansible.builtin.command:
        cmd: cat /proc/meminfo
      register: memory_info
      changed_when: false

    - name: AI report
      ai:
        text: "メモリーの使用率についてMarkdown形式でレポートを作成してください。会話は不要です。あなたはAPIのようにレポートの結果だけを出力してください。 {{ memory_info.stdout }}"
        # api_key: APIキーを指定または、環境変数 AI_API_KEY を設定
        # model: モデルを指定または、環境変数 AI_MODEL を設定
      register: result

    - name: Output report
      ansible.builtin.copy:
        content: "{{ result.message }}"
        dest: ../ai_report.md
        mode: "0644"
      when: result.message is defined
```

作成されたレポートです。それっぽいのが出てきましたね！

```md:../ai_report.md
# メモリー使用率レポート

## 概要

このレポートは、提供されたシステム情報に基づくメモリー使用率の分析結果です。

### 全体的な状態

*   **総メモリー**: 15.50 GB (16257360 kB)
*   **利用可能メモリー**: 10.28 GB (10776732 kB)
*   **使用済みメモリー**: 6.35 GB (6663376 kB)
*   **使用率**: 41.0%
*   **スワップ領域**: 4.00 GB (4194304 kB)
*   **スワップ使用率**: 0.0%

## メインメモリー使用状況

| 指標                | 量 (kB)    | 量 (GB)  | 割合 (%) | 備考                               |
| :------------------ | :--------- | :------- | :------- | :--------------------------------- |
| **総メモリー**      | 16257360   | 15.50    | 100.0    | システムが利用可能な物理メモリーの総量 |
| **空きメモリー**    | 9593984    | 9.15     | 59.0     | プロセスが現在使用していないメモリー       |
| **利用可能メモリー** | 10776732   | 10.28    | 66.3     | 新しいアプリケーションが即座に利用できると見積もられるメモリー量 (キャッシュを含む) |
| **使用済みメモリー** | 6663376    | 6.35     | 41.0     | システムおよびアプリケーションによって使用されているメモリー |
| **バッファ/キャッシュ** | 1367568    | 1.30     | 8.4      | ファイルシステムキャッシュ、ディスクバッファなど |

*   **計算方法**:
    *   `使用済みメモリー` = `MemTotal` - `MemFree`
    *   `バッファ/キャッシュ` = `Buffers` + `Cached`
    *   `使用率` = (`使用済みメモリー` / `MemTotal`) * 100

## スワップ領域使用状況

| 指標               | 量 (kB)    | 量 (GB)  | 割合 (%) | 備考                                   |
| :----------------- | :--------- | :------- | :------- | :------------------------------------- |
| **スワップ合計**   | 4194304    | 4.00     | 100.0    | スワップ領域の総量                         |
| **スワップ空き**   | 4194304    | 4.00     | 100.0    | 現在使用されていないスワップ領域の量     |
| **スワップ使用済み** | 0          | 0.00     | 0.0      | システムによって使用されているスワップ領域の量 |

*   **計算方法**:
    *   `スワップ使用済み` = `SwapTotal` - `SwapFree`
    *   `スワップ使用率` = (`スワップ使用済み` / `SwapTotal`) * 100

## 詳細ブレイクダウン (抜粋)

| 指標             | 量 (kB)   | 備考                                                                        |
| :--------------- | :-------- | :-------------------------------------------------------------------------- |
| `Active`         | 628728    | 最近使用され、RAMに残っている可能性が高いメモリー量                         |
| `Inactive`       | 5421508   | 最近使用されていないが、まだRAMに残っているメモリー量 (解放される可能性あり) |
| `AnonPages`      | 4589796   | ファイルにマップされていない、匿名のページ (通常はプログラムデータ)         |
| `PageTables`     | 86548     | 仮想メモリーから物理メモリーへのマッピングを保持するテーブルに費やされるメモリー |
| `KernelStack`    | 13072     | カーネルプロセスが使用するスタックメモリー                                  |
| `Slab`           | 222452    | カーネルがオブジェクトをキャッシュするために使用するメモリー              |
| `KReclaimable`   | 133568    | 再利用可能なSlabメモリー                                                    |
| `Committed_AS`   | 5380308   | 現在のシステムが約束した仮想メモリーの量                                  |

## 結論

システムは豊富な利用可能メモリー（総メモリーの約66%）を保持しており、スワップ領域は全く使用されていません。これは、現在のメモリー需要が物理RAMで十分に満たされていることを示しており、メモリーリソースに関しては健全な状態にあると言えます。
```

# あとがき

なんでもいいので生成AIに絡んだ記事を書いてみたいという下心で書きました。

Ansibleモジュールをはじめとした各種プラグインの開発をできるようになるとAnsibleの世界がぐっと広がります。既存のモジュールで実現できないかを考えてそれでもいいのが見つからない...となれば自分で作ってみるのも1つの手です。

# 参考

- [Ansible project contributors. "Developer Guide". Ansible Community Documentation.](https://docs.ansible.com/ansible/latest/dev_guide/index.html)
- [Ansible project contributors. "Ansible module architecture". Ansible Community Documentation.](https://docs.ansible.com/ansible/latest/dev_guide/developing_program_flow_modules.html)
- [Google LLC. "Gemini Developer API". Google AI for Developers.](https://ai.google.dev/gemini-api/docs)
- [Python Software Foundation. "urllib — URL handling modules". Python.org.](https://docs.python.org/3.13/library/urllib.html)

[^Introduction-to-modules]: Ansible project contributors. "Introduction to modules". Ansible Community Documentation. 2025-8-21. https://docs.ansible.com/ansible/latest/module_plugin_guide/modules_intro.html, (2025-08-24)

[^Adding-modules-and-plugins-locally]: Ansible project contributors. "Adding modules and plugins locally". Ansible Community Documentation. 2025-8-21. https://docs.ansible.com/ansible/latest/dev_guide/developing_locally.html, (2025-08-24)

[^default-module-path]: Ansible project contributors. "Ansible Configuration Settings". Ansible Community Documentation. 2025-8-21. https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-module-path, (2025-08-24)

[^python-shebang-utf-8-coding]: Ansible project contributors. "Module format and documentation". Ansible Community Documentation. 2025-8-21. https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_documenting.html#python-shebang-utf-8-coding, (2025-08-24)
