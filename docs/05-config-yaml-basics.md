---
title: "第5章 config.yaml の基本構造"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - config
  - yaml
  - models
---

## この章で学ぶこと

- `config.yaml` の役割と、ファイルがどこに置かれているかを理解する
- YAML の基本的な書き方（インデント・データ型・リスト）を把握する
- `config.yaml` のトップレベルキーの構成を俯瞰する
- ロール（Chat 用・Autocomplete 用）ごとに異なるモデルを割り当てる考え方を学ぶ
- Ollama・vLLM・LM Studio それぞれのエンドポイント記述例を確認する

---

## config.yaml とは何か

Continue は、動作に関するほぼすべての設定を `config.yaml` という 1 つのファイルで管理しています。「どのモデルを使うか」「どのような補完を行うか」「どんなコンテキストを参照するか」——これらをすべて `config.yaml` に記述することで、チームメンバーと同一の設定を共有したり、プロジェクトごとに設定を切り替えたりすることができます。

エアギャップ環境では外部サービスと自動連携することができないため、ローカル LLM への接続情報を `config.yaml` に正確に記述することが、Continue を正しく動かすための最重要ステップです。この章では、ファイルの場所と構造を理解し、実際にローカル LLM へ接続できる最小限の設定を書けるようになることを目標とします。

---

## ファイルの場所と開き方

### デフォルトのファイルパス

`config.yaml` はユーザーのホームディレクトリ配下の `.continue/` フォルダに置かれます。OS ごとのデフォルトパスは次のとおりです。

| OS | パス |
| --- | --- |
| Windows | `C:\Users\<ユーザー名>\.continue\config.yaml` |
| macOS | `~/.continue/config.yaml` |
| Linux | `~/.continue/config.yaml` |

`<ユーザー名>` の部分は実際のログインユーザー名に読み替えてください。

!!! note "プロジェクト固有の設定について"
    `.continue/` フォルダをプロジェクトのルートディレクトリに置くと、そのプロジェクト専用の設定として機能します。プロジェクト固有の設定はユーザー設定より優先されます。この機能は第 11 章で詳しく扱います。

### VS Code から開く方法

VS Code のサイドバーに表示される Continue のアイコンをクリックし、チャットパネル右上の歯車アイコン（⚙）を選択すると `config.yaml` が直接エディタで開きます。ファイルを手動で探す手間が省けるため、この方法を推奨します。

ターミナルから開きたい場合は、次のコマンドでファイルの存在を確認できます。

```bash
# Linux / macOS
ls -la ~/.continue/config.yaml

# Windows PowerShell
Test-Path "$env:USERPROFILE\.continue\config.yaml"
```

!!! warning "エアギャップ環境での注意"
    Continue の初回起動時にデフォルト設定がインターネットから取得されることがあります。エアギャップ環境では、システム管理者があらかじめ `config.yaml` のテンプレートをオフラインで配布しておくことを推奨します。詳細は第 3 章を参照してください。

---

## YAML 構文の基礎

`config.yaml` は YAML（YAML Ain't Markup Language）形式で記述します。YAML は設定ファイルとして広く使われているフォーマットで、人間が読み書きしやすいことが特長です。ここでは `config.yaml` を書くうえで必要な最低限の文法を説明します。

### インデントが構造を決める

YAML では **半角スペースによるインデント** が階層構造を表します。タブ文字（Tab キー）は使用できません。VS Code のデフォルト設定ではタブが自動的にスペースに変換されるため、通常は意識しなくて構いませんが、外部エディタで編集する場合は注意してください。

```yaml
# 正しい例（スペース 2 つでインデント）
models:
  - name: my-chat-model
    provider: ollama
    model: llama3
```

```yaml
# 誤った例（タブ文字を使用）
models:
	- name: my-chat-model   # ← タブはエラーになる
```

### 主なデータ型

`config.yaml` でよく使うデータ型を以下に示します。

```yaml
# 文字列（クォートなしでも書けるが、コロンやスペースを含む場合はクォートで囲む）
name: my-continue-config
apiBase: "http://localhost:11434"

# 真偽値
enabled: true

# リスト（ハイフン + スペースで始める）
tags:
  - continue
  - vscode
  - on-premise

# ネストしたオブジェクト（インデントで階層を表す）
models:
  - name: my-model
    provider: ollama
    model: llama3
```

### コメント

`#` より後ろの文字はコメントとして扱われ、Continue の動作には影響しません。設定の意図を残すために積極的に活用してください。

```yaml
models:
  - name: Llama 3 (Chat)    # チャット用モデル
    provider: ollama
    model: llama3            # Ollama にプルされているモデル名と一致させる
```

!!! tip "YAML のバリデーション"
    `config.yaml` を保存した後に Continue が正しく動作しない場合、まず YAML の構文エラーを疑いましょう。VS Code には YAML 言語サーバが組み込まれており、赤い波線でエラー箇所を示してくれます。

---

## config.yaml の全体構造を俯瞰する

現時点の Continue の `config.yaml` は、大きく次のトップレベルキーで構成されています。この章では各キーの概要を把握することを目標とし、詳細な設定方法は後続の章で順に扱います。

| キー | 役割 | 詳細を扱う章 |
| --- | --- | --- |
| `name` | 設定ファイルの名前（必須） | 本章 |
| `version` | 設定ファイルのバージョン（必須） | 本章 |
| `schema` | スキーマバージョン（必須、値は `v1`） | 本章 |
| `models` | Chat・Edit・Autocomplete・Embed など用途ごとのモデルの一覧（`roles:` で用途を指定） | 本章・第 6・7 章 |
| `context` | `@file` や `@codebase` などのコンテキスト機能（各エントリは `provider:` キーを使う） | 第 6 章・第 10 章 |
| `rules` | モデルへの共通指示（コーディング規約など） | 第 11 章 |
| `prompts` | 独自に定義するカスタムプロンプト | 第 11 章 |
| `docs` | ドキュメントサイトのインデックス設定 | 第 10 章 |
| `mcpServers` | MCP サーバの接続設定 | 第 12 章 |

以下は、最小限の要素だけを記述した `config.yaml` の全体像です。実際の設定はこの雛形をベースに拡張していきます。

```yaml
# ~/.continue/config.yaml  ——  最小構成の例

name: my-continue-config
version: 1.0.0
schema: v1

models:
  - name: My Chat Model
    provider: ollama
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit

  - name: My Autocomplete Model
    provider: ollama
    model: <your-autocomplete-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - autocomplete

context:
  - provider: code
  - provider: docs
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: codebase
```

!!! note "本構成では使用しません"
    Continue には `embeddingsProvider`（埋め込みモデル）や `reranker`（再ランキング）を指定するキーもありますが、クラウドサービス前提の機能が多く含まれます。エアギャップ環境でのローカル Embedding モデル活用については第 10 章で改めて扱います。

---

## ロール別モデル割り当ての考え方

Continue は「Chat」「Autocomplete」という用途ごとに、異なるモデルを指定できます。これはエアギャップ環境においても重要な設計判断です。

### なぜ用途でモデルを分けるのか

Chat では、コードの意味を理解し、設計方針を説明し、複数ファイルにわたる質問に答えるために、**文脈理解力の高い大きなモデル**が向いています。一方、Autocomplete では、キーを押すたびにリアルタイムで補完候補を生成するため、**レスポンスが速い小さなモデル**が向いています。

たとえば、次のような組み合わせが典型的です。

| 用途 | モデルの特性 | 例 |
| --- | --- | --- |
| Chat / Edit / Agent | 高精度・大パラメータ | Llama 3 70B、Qwen 2.5 72B など |
| Autocomplete | 低レイテンシ・小パラメータ | Qwen 2.5 Coder 1.5B、DeepSeek Coder 1.3B など |

### `models` キーと `roles` フィールド

`config.yaml` では、Chat・Edit・Agent で使うモデルも Autocomplete 専用モデルも、すべて `models` リストに記述します。用途の違いは各エントリの `roles:` フィールドで指定します。

```yaml
# Chat / Edit 用（複数登録して UI で切り替え可能）
models:
  - name: Llama 3 70B（大型）
    provider: ollama
    model: llama3:70b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit

  - name: Qwen 2.5 7B（中型・軽量）
    provider: ollama
    model: qwen2.5:7b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat

  # Autocomplete 専用（roles: [autocomplete] で指定）
  - name: Qwen 2.5 Coder 1.5B（補完専用）
    provider: ollama
    model: qwen2.5-coder:1.5b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - autocomplete
```

`models` リストには複数のモデルを登録でき、VS Code の Continue パネルからドロップダウンで切り替えられます。大型モデルと中型モデルを両方登録しておき、作業内容に応じて使い分けるのが実用的です。

!!! tip "Autocomplete モデルの選び方"
    Autocomplete モデルはコード補完に特化した小型モデルを選ぶと、レスポンスが速く快適に使えます。`qwen2.5-coder`（1.5B〜7B）や `deepseek-coder`（1.3B）などがエアギャップ環境でよく使われます。`roles: [autocomplete]` を指定して `models` リストに追加します。詳しくは第 7 章で扱います。

---

## ローカル LLM エンドポイントの設定例

ここでは、よく使われるローカル LLM サーバ 3 種類（Ollama・vLLM・LM Studio）それぞれの最小限の接続設定を示します。いずれも LLM サーバは別途オフラインでセットアップ済みであることを前提とします。

### Ollama を使う場合

Ollama（オラマ）はコマンド 1 つでローカル LLM を起動できるツールで、エアギャップ環境でも広く使われています。`provider: ollama` を指定し、Ollama が動作しているホストの URL を `apiBase` に記述します。

```yaml
models:
  - name: Llama 3（Ollama）
    provider: ollama
    model: <your-model-name>          # 例: llama3、qwen2.5:7b
    apiBase: "http://<your-internal-llm-host>:11434"
                                      # Ollama のデフォルトポートは 11434
    roles:
      - chat
      - edit

  - name: Qwen 2.5 Coder（Ollama）
    provider: ollama
    model: <your-autocomplete-model-name>   # 例: qwen2.5-coder:1.5b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - autocomplete
```

`model` の値は、Ollama にプルされているモデル名と完全に一致している必要があります。社内の Ollama サーバで利用可能なモデル一覧は、サーバ管理者に確認してください。

!!! note "同一マシン上で Ollama を動かす場合"
    Ollama を VS Code と同じマシンで起動する場合は `apiBase: "http://localhost:11434"` と記述します。別サーバで動かしている場合は、そのホスト名または IP アドレスに置き換えてください。

### vLLM を使う場合

vLLM（ブイエルエルエム）は高スループットな LLM 推論エンジンです。OpenAI 互換の API を提供するため、Continue では `provider: openai` として設定し、エンドポイントを社内の vLLM サーバに向けます。

```yaml
models:
  - name: Qwen 2.5 72B（vLLM）
    provider: openai          # vLLM は OpenAI 互換 API を提供するため openai を指定
    model: <your-model-name>  # vLLM にデプロイされているモデル名
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
                              # vLLM のデフォルトポートは 8000、パスは /v1
    apiKey: "dummy"           # vLLM は認証不要な構成もあるが、フィールド自体は必要な場合がある
    roles:
      - chat
      - edit

  - name: DeepSeek Coder（vLLM）
    provider: openai
    model: <your-autocomplete-model-name>
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - autocomplete
```

!!! warning "apiKey の扱い"
    vLLM を認証なしで運用している場合でも、`apiKey` フィールドに任意の文字列（`"dummy"` など）を記述しないとエラーになる構成があります。社内の vLLM サーバの設定に合わせて調整してください。

### LM Studio を使う場合

LM Studio（エルエムスタジオ）は GUI でローカル LLM を管理・実行できるツールです。内部で OpenAI 互換サーバを立ち上げるため、vLLM と同様に `provider: openai` を使います。LM Studio のデフォルトポートは `1234` です。

```yaml
models:
  - name: Llama 3（LM Studio）
    provider: lmstudio        # LM Studio 専用プロバイダも選択可能
    model: <your-model-name>  # LM Studio でロードされているモデル名
    apiBase: "http://localhost:1234"
                              # LM Studio のデフォルトポートは 1234
    roles:
      - chat
      - edit
```

`provider: lmstudio` を使うと Continue 側でデフォルトポート（`1234`）が自動的に適用されるため、`apiBase` の記述を省略できます。ただし、LM Studio を別のマシンで動かしている場合や、ポートを変更している場合は `apiBase` を明示的に指定してください。

```yaml
  - name: Phi-3 Mini（LM Studio）
    provider: lmstudio
    model: <your-autocomplete-model-name>
    apiBase: "http://<your-internal-llm-host>:1234"
    roles:
      - autocomplete
```

!!! note "本構成では使用しません"
    LM Studio にはクラウド経由でモデルをダウンロードする機能がありますが、エアギャップ環境では使用しません。モデルファイル（`.gguf` 形式）は社内のファイルサーバ等からオフラインで入手し、LM Studio にインポートして使用してください。

---

## 設定を保存して動作を確認する

### 保存と自動リロード

`config.yaml` を保存すると、Continue は設定を**自動的にリロード**します。VS Code を再起動する必要はありません。保存の瞬間にサイドバーの Continue パネルが一瞬リフレッシュするのが確認の目安です。

### モデルが切り替わったことを確認する

Continue のチャットパネル上部にあるモデル選択ドロップダウンを開き、`config.yaml` に記述したモデルの名前（`name` キーの値）が表示されているかを確認します。表示されていれば設定の読み込みは成功しています。

### 接続の動作確認

チャット入力欄に短いメッセージ（例: 「こんにちは」）を送信して、モデルから応答が返ってくることを確認してください。エアギャップ環境では、LLM サーバへのネットワーク到達性がすべての前提となります。応答が返ってこない場合は、次の点を確認してください。

1. **LLM サーバが起動しているか** — サーバ側のプロセスやサービスが動作しているかを確認します
2. **`apiBase` の URL が正しいか** — ホスト名、ポート番号、パス（`/v1` の有無など）を見直します
3. **ネットワーク到達性があるか** — `curl` などで LLM サーバの疎通を確認します

```bash
# Ollama の場合（稼働確認）
curl http://<your-internal-llm-host>:11434/api/tags

# vLLM / LM Studio（OpenAI 互換）の場合
curl http://<your-internal-llm-host>:<port>/v1/models
```

疎通は取れているのに Continue から応答がない場合は、第 14 章のトラブルシューティングを参照してください。

!!! tip "YAML 構文エラーの確認"
    保存後にモデルがドロップダウンに表示されない場合、YAML の構文エラーが原因の可能性があります。VS Code のエラーパネル（`表示` → `問題`）を開いて確認してみてください。

---

## まとめ

- `config.yaml` は Continue の動作を制御する唯一の設定ファイルで、`~/.continue/` 配下に置かれる
- YAML はインデントに意味があるため、スペース数を揃え、タブ文字を使わないことが重要
- `models` キーに `roles:` を指定することで、Chat・Edit・Agent 用と Autocomplete 専用のモデルを使い分ける
- Ollama は `provider: ollama`、vLLM と LM Studio は `provider: openai` または `provider: lmstudio` を使い、それぞれ `apiBase` でローカルエンドポイントを指定する
- ファイルを保存すると設定が即座にリロードされ、チャットパネルのドロップダウンに反映される

---

## 次の章へ

次は [第 6 章 Chat 機能の基本](06-chat-basics.md) で、設定した LLM を実際に使ってコードについて質問し、AI コーディングアシスタントとしての Continue を使いこなす方法を学びます。

---

## 参考リンク

- [Continue 公式ドキュメント — Configuration Overview](https://docs.continue.dev/reference/config)
- [Continue 公式ドキュメント — Model Providers](https://docs.continue.dev/reference/model-providers/ollama)
- [YAML 公式仕様](https://yaml.org/spec/1.2.2/)
- [MkDocs Material — Admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/)
