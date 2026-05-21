---
title: "付録A config.yaml リファレンス"
type: appendix
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - config
  - yaml
  - reference
---

## このリファレンスの使い方

本付録は、Continue の設定ファイル `config.yaml` に記述できる全キー・全オプションを網羅した辞書的リファレンスです。チュートリアル形式で最小構成を学ぶ [第 5 章](../05-config-yaml-basics.md) の補足として、手元に置いて引くことを想定しています。

各節には「詳細は第 N 章」という相互参照を記載しており、設定の背景や操作手順は対応する章を参照してください。エアギャップ環境では使用できない機能には「**本構成では使用しません**」と明示しています。

---

## トップレベルキー早見表

`config.yaml` に記述できる全トップレベルキーを以下に示します。

| キー | 型 | 必須 | 概要 | 詳細 |
| --- | --- | --- | --- | --- |
| `models` | リスト | ✓ | Chat・Edit・Agent で使うモデルの一覧 | [models](#models) / 第 5・6 章 |
| `tabAutocompleteModel` | オブジェクト | — | Autocomplete（タブ補完）専用モデル | [tabAutocompleteModel](#tabautocompletemodel) / 第 7 章 |
| `tabAutocompleteOptions` | オブジェクト | — | Autocomplete の詳細チューニング | [tabAutocompleteOptions](#tabAutocompleteoptions) / 第 7 章 |
| `contextProviders` | リスト | — | `@file` / `@codebase` などの参照機能 | [contextProviders](#contextproviders) / 第 6・10 章 |
| `slashCommands` | リスト | — | `/edit` などのスラッシュコマンド | [slashCommands](#slashcommands) / 第 11 章 |
| `customCommands` | リスト | — | ユーザー定義のカスタムコマンド | [customCommands](#customcommands) / 第 11 章 |
| `rules` | リスト | — | モデルへの共通指示（コーディング規約など） | [rules](#rules) / 第 11 章 |
| `models`（`roles: [embed]`） | リスト | — | ローカル Embedding モデルの設定（`models` リストに `roles: [embed]` で登録） | [models](#models) / 第 10 章 |
| `mcpServers` | リスト | — | MCP サーバの接続設定 | [mcpServers](#mcpservers) / 第 12 章 |
| `allowAnonymousTelemetry` | 真偽値 | — | テレメトリの送信可否（デフォルト: `true`） | [プライバシー・テレメトリ設定](#プライバシーテレメトリ設定) / 第 4 章 |

---

## models

Chat・Edit・Agent で使うモデルを登録するリストです。複数のモデルを並べて登録でき、VS Code の Continue パネルのドロップダウンから切り替えられます。

### 共通フィールド一覧

| フィールド | 型 | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- | --- |
| `name` | 文字列 | ✓ | — | VS Code の UI に表示されるモデルの表示名。任意の文字列を指定できます |
| `provider` | 文字列 | ✓ | — | 接続するバックエンドの種別。`ollama` / `openai` / `lmstudio` など |
| `model` | 文字列 | ✓ | — | バックエンドに登録されているモデル名。プロバイダ側の識別子と一致させます |
| `apiBase` | 文字列 | — | プロバイダ依存 | LLM サーバのエンドポイント URL。プロバイダのデフォルトから変更する場合に指定します |
| `api_key` | 文字列 | — | — | 認証トークン。社内サーバで認証が不要でも、プロバイダによっては任意の文字列が必要な場合があります |
| `title` | 文字列 | — | `name` と同値 | スラッシュコマンド等での参照に使う短い識別子 |
| `contextLength` | 整数 | — | プロバイダ依存 | モデルが受け付ける最大コンテキスト長（トークン数）。指定しない場合はプロバイダのデフォルト値が適用されます |
| `maxStopWords` | 整数 | — | プロバイダ依存 | 停止ワードの最大数 |
| `promptTemplates` | オブジェクト | — | — | `chat` / `edit` などのプロンプトテンプレートを上書きする場合に指定します |

### プロバイダ別設定

#### ollama

Ollama のデフォルトポートは `11434`、`apiBase` を省略すると `http://localhost:11434` が使われます。

```yaml
models:
  - name: Llama 3（Ollama）
    provider: ollama
    model: <your-model-name>         # 例: llama3、qwen2.5:7b
    apiBase: "http://<your-internal-llm-host>:11434"
```

| 固有フィールド | 型 | 説明 |
| --- | --- | --- |
| `apiBase` | 文字列 | Ollama サーバの URL。別ホストで動かす場合に指定します |

#### openai（vLLM / LM Studio OpenAI 互換モード含む）

vLLM や LM Studio の OpenAI 互換 API にも `provider: openai` を使います。`apiBase` でエンドポイントを社内サーバに向けてください。

```yaml
models:
  - name: Qwen 2.5 72B（vLLM）
    provider: openai
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    api_key: "dummy"                 # 認証不要でもフィールドが必要な場合があります
```

| 固有フィールド | 型 | 説明 |
| --- | --- | --- |
| `apiBase` | 文字列 | OpenAI 互換サーバの URL。末尾 `/v1` まで含めて指定します |
| `api_key` | 文字列 | API キー。認証不要な構成でも空白以外の文字列が必要な場合があります |

!!! warning "vLLM の api_key"
    vLLM を認証なしで運用している場合でも、`api_key` フィールドに `"dummy"` など任意の文字列を設定しないとエラーになる構成があります。社内サーバの設定に合わせて調整してください。

#### lmstudio

LM Studio 専用プロバイダです。デフォルトポート `1234` が自動で適用されるため、同一マシンで動かす場合は `apiBase` を省略できます。

```yaml
models:
  - name: Llama 3（LM Studio）
    provider: lmstudio
    model: <your-model-name>
    # api_base を省略すると http://localhost:1234 が使われます
    # 別ホストで動かす場合は明示してください
    apiBase: "http://<your-internal-llm-host>:1234"
```

### 複数モデルの登録と切り替え

`models` リストに複数エントリを並べると、VS Code の Continue パネルのドロップダウンで切り替えられます。用途（高精度タスク・軽量タスク）に応じてモデルを使い分けるのが実用的です。

```yaml
models:
  - name: Llama 3 70B（高精度）
    provider: ollama
    model: llama3:70b
    apiBase: "http://<your-internal-llm-host>:11434"

  - name: Qwen 2.5 7B（軽量）
    provider: ollama
    model: qwen2.5:7b
    apiBase: "http://<your-internal-llm-host>:11434"
```

---

## tabAutocompleteModel

Autocomplete（タブ補完）専用モデルを 1 つだけ指定します。`models` リストとは独立しており、レスポンス速度を優先した小型モデルを選ぶのが一般的です（→ 詳細は [第 7 章](../07-autocomplete.md)）。

### フィールド一覧

`models` の共通フィールドをすべて使用できます。加えて以下の Autocomplete 固有フィールドが利用可能です。

| フィールド | 型 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `name` | 文字列 | — | 表示名（必須） |
| `provider` | 文字列 | — | プロバイダ識別子（必須） |
| `model` | 文字列 | — | モデル名（必須） |
| `apiBase` | 文字列 | プロバイダ依存 | エンドポイント URL |

```yaml
tabAutocompleteModel:
  name: Qwen 2.5 Coder 1.5B（補完専用）
  provider: ollama
  model: qwen2.5-coder:1.5b
  apiBase: "http://<your-internal-llm-host>:11434"
```

### tabAutocompleteOptions（詳細チューニング）

補完品質・レイテンシを調整するオプションです。`tabAutocompleteOptions` キーをトップレベルに追加して設定します。

| フィールド | 型 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `useCopyBuffer` | 真偽値 | `false` | コピーバッファの内容をコンテキストに含めるかどうか |
| `useFileSuffix` | 真偽値 | `true` | カーソル後方のコードをコンテキストに含めるかどうか（FIM 対応モデル向け） |
| `maxPromptTokens` | 整数 | `1024` | プロンプトに含める最大トークン数。増やすと精度が上がる場合がありますが、レイテンシも増加します |
| `prefixPercentage` | 数値 | `0.85` | プロンプトのうちカーソル前方に割り当てる割合（0〜1） |
| `maxSuffixPercentage` | 数値 | `0.25` | プロンプトのうちカーソル後方に割り当てる最大割合（0〜1） |
| `debounceDelay` | 整数（ms） | `350` | キー入力から補完リクエスト送信までの待機時間。低速なモデルでは大きくするとリクエストが減ります |
| `multilineCompletions` | 文字列 | `"auto"` | 複数行補完の動作。`"always"` / `"never"` / `"auto"` のいずれか |
| `disableInFiles` | リスト | `[]` | 補完を無効にするファイルのグロブパターン一覧（例: `["*.md", "*.txt"]`） |

```yaml
tabAutocompleteOptions:
  maxPromptTokens: 512       # 低速環境ではトークン数を減らしてレイテンシを改善
  debounceDelay: 500
  multilineCompletions: "auto"
  disableInFiles:
    - "*.md"
    - "*.txt"
```

!!! tip "レイテンシが気になる場合"
    `debounceDelay` を増やす・`maxPromptTokens` を減らす・`multilineCompletions: "never"` にする、の 3 つがレイテンシ改善の基本アプローチです。詳細は [第 7 章](../07-autocomplete.md) を参照してください。

---

## contextProviders

`@file`・`@codebase` などの参照機能（コンテキストプロバイダ）を有効化するリストです（→ 詳細は [第 6 章](../06-chat-basics.md) / [第 10 章](../10-internal-models-context.md)）。

### 組み込みコンテキストプロバイダ一覧

| name | 呼び出し方 | 概要 | エアギャップ対応 |
| --- | --- | --- | --- |
| `code` | `@code` | 関数・クラス等のシンボルを検索して参照 | ✓ |
| `codebase` | `@codebase` | コードベース全体をインデックスして参照 | ✓（ローカル Embedding 必須）|
| `docs` | `@docs` | ドキュメントサイトをインデックスして参照 | ✓（ローカルで事前取得が必要）|
| `diff` | `@diff` | 現在の Git diff を参照 | ✓ |
| `terminal` | `@terminal` | ターミナルの出力を参照 | ✓ |
| `problems` | `@problems` | VS Code の「問題」パネルの内容を参照 | ✓ |
| `folder` | `@folder` | 指定フォルダ内のファイルを参照 | ✓ |
| `file` | `@file` | 特定のファイルを参照 | ✓ |
| `open` | `@open` | VS Code で開いているファイル一覧を参照 | ✓ |
| `search` | `@search` | リポジトリ内をキーワード検索して参照 | ✓ |
| `url` | `@url` | 指定 URL の内容を参照 | △（社内 URL のみ可） |
| `tree` | `@tree` | ディレクトリツリーを参照 | ✓ |

### 各プロバイダの設定フィールド

#### codebase

コードベース全体をベクトル検索で参照します。ローカル Embedding モデル（`embeddingsProvider`）の設定が必須です。

```yaml
contextProviders:
  - name: codebase
```

設定フィールドはなく、`name: codebase` を追加するだけで有効になります。インデックス構築時間はリポジトリの規模に依存します。

#### docs

ドキュメントサイトをクローリングしてインデックス化します。エアギャップ環境では、外部 URL はアクセスできないため、**社内で配信しているドキュメントサイトの URL のみ**指定してください。

```yaml
contextProviders:
  - name: docs
```

初回利用時に VS Code の Continue パネルから「Add Doc」でサイト URL を登録します。

!!! warning "エアギャップ環境での制限"
    外部のドキュメントサイト（例: 公式ドキュメントサイト）は、エアギャップ環境からアクセスできないため `@docs` でインデックス化できません。社内イントラネットで配信しているサイトのみ対象となります。

#### folder

```yaml
contextProviders:
  - name: folder
```

チャット入力で `@folder` と入力すると、参照したいフォルダをインタラクティブに選択できます。

#### file

```yaml
contextProviders:
  - name: file
```

`@file` と入力するとファイル名のオートコンプリートが起動します。

#### url

```yaml
contextProviders:
  - name: url
```

`@url` に続けて URL を指定すると、そのページの内容が取得されてコンテキストに含まれます。エアギャップ環境では、ファイアウォール等で許可された社内 URL にのみ使用してください。

---

## slashCommands

`/edit`・`/comment` などのスラッシュコマンドを有効化するリストです（→ 詳細は [第 11 章](../11-customization.md)）。

### 組み込みスラッシュコマンド一覧

| name | 動作概要 |
| --- | --- |
| `edit` | 選択中のコードを指示に従って書き換えます |
| `comment` | 選択中のコードにコメントを追加します |
| `share` | 現在の会話をマークダウンとしてエクスポートします |
| `cmd` | 自然言語からシェルコマンドを生成します |
| `commit` | Git の変更をもとにコミットメッセージを生成します |

```yaml
slashCommands:
  - name: edit
    description: "選択範囲のコードを編集する"
  - name: comment
    description: "コードにコメントを追加する"
  - name: share
    description: "会話をマークダウンとしてエクスポートする"
  - name: cmd
    description: "シェルコマンドを生成する"
  - name: commit
    description: "コミットメッセージを生成する"
```

### カスタムスラッシュコマンドのフィールド

`slashCommands` リストには、組み込みコマンド以外にカスタムコマンドも追加できます。

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `name` | 文字列 | ✓ | コマンド名。`/` に続けて入力する識別子（例: `refactor` → `/refactor`） |
| `description` | 文字列 | ✓ | コマンド一覧に表示される説明文 |

---

## customCommands

`slashCommands` の組み込みコマンドとは異なり、プロンプト本文を直接 `config.yaml` に記述して独自コマンドを定義します（→ 詳細は [第 11 章](../11-customization.md)）。

### フィールド一覧

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `name` | 文字列 | ✓ | コマンド名。`/` に続けて入力する識別子 |
| `description` | 文字列 | ✓ | コマンド一覧に表示される説明文 |
| `prompt` | 文字列 | ✓ | モデルに送信するプロンプト本文。`{{{ input }}}` で追加の入力を受け取れます |

```yaml
customCommands:
  - name: review
    description: "コードをセキュリティ観点でレビューする"
    prompt: >-
      次のコードについて、セキュリティ上の問題点を日本語で指摘してください。
      問題がない場合は「問題なし」と答えてください。

      {{{ input }}}
```

---

## rules

モデルへの共通指示を記述するリストです。すべての会話・編集・エージェント操作に自動的に付与されます。社内コーディング規約や回答フォーマットの指定に活用してください（→ 詳細は [第 11 章](../11-customization.md)）。

### フィールド一覧

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `name` | 文字列 | ✓ | ルールの識別名 |
| `rule` | 文字列 | ✓ | モデルに渡す指示の本文 |
| `globs` | リスト | — | このルールを適用するファイルのグロブパターン。省略すると全ファイルに適用されます |

```yaml
rules:
  - name: 日本語で回答する
    rule: "回答は必ず日本語で行ってください。コードのコメントも日本語で記述してください。"

  - name: Python スタイルガイド
    rule: >-
      Python コードを書く際は以下の規約に従ってください：
      - 型ヒントを必ず付与する
      - docstring は Google スタイルで記述する
      - 1 行は 100 文字以内に収める
    globs:
      - "**/*.py"
```

---

## Embedding モデルの設定（roles: [embed]）

`@codebase` や `@docs` 機能でコードベースをベクトル検索するために使うローカル Embedding モデルは、`models` リストにエントリを追加し、`roles` フィールドに `embed` を指定して登録します（→ 詳細は [第 10 章](../10-internal-models-context.md)）。

### フィールド一覧

`models` の共通フィールド（`name`・`provider`・`model`・`apiBase` など）に加え、次のフィールドを指定します。

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `roles` | リスト | ✓ | `[embed]` を指定。Embedding 専用として登録されます |

### ローカル Embedding 対応プロバイダ

#### ollama

```yaml
models:
  - name: Embeddings（nomic-embed-text）
    provider: ollama
    model: nomic-embed-text     # Ollama にプルされている Embedding モデル名
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - embed
```

Ollama で利用できる Embedding モデルの例として `nomic-embed-text`・`mxbai-embed-large` などがあります。社内 Ollama サーバに用意されているモデルを管理者に確認してください。

#### openai 互換（vLLM など）

```yaml
models:
  - name: Embeddings（bge-m3 on vLLM）
    provider: openai
    model: <your-embedding-model-name>
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    api_key: "dummy"
    roles:
      - embed
```

!!! note "本構成では使用しません"
    クラウドの OpenAI Embeddings API（`text-embedding-ada-002` など）はエアギャップ環境では使用できません。必ずローカルで動作する Embedding モデルを指定してください。

---

## mcpServers

MCP（Model Context Protocol）サーバへの接続を設定するリストです（→ 詳細は [第 12 章](../12-mcp-onprem.md)）。ローカルまたは社内で動作する MCP サーバのみ接続してください。

### フィールド一覧

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `name` | 文字列 | ✓ | サーバの識別名。Continue 内での表示名として使われます |
| `command` | 文字列 | △ | MCP サーバを起動するコマンド（stdio 方式の場合） |
| `args` | リスト | — | コマンドに渡す引数 |
| `env` | オブジェクト | — | 環境変数のキー・バリューペア |
| `url` | 文字列 | △ | MCP サーバの SSE エンドポイント URL（SSE 方式の場合） |

#### stdio 方式

ローカルプロセスとして MCP サーバを起動する方式です。エアギャップ環境では最も一般的な接続方式です。

```yaml
mcpServers:
  - name: filesystem
    command: node
    args:
      - "/opt/mcp-servers/filesystem/index.js"
      - "--root"
      - "/home/<your-username>/projects"
```

#### SSE 方式

HTTP SSE（Server-Sent Events）で接続する方式です。別ホストで動作する MCP サーバに接続する場合に使います。

```yaml
mcpServers:
  - name: internal-git
    url: "http://<your-internal-mcp-host>:<port>/sse"
```

---

## プライバシー・テレメトリ設定

セキュリティ・監査に関わる設定キーをまとめます（→ 詳細は [第 4 章](../04-telemetry-airgap-verification.md) / [第 13 章](../13-security-operations.md)）。

### allowAnonymousTelemetry

Continue が匿名の使用統計を外部サーバへ送信するかどうかを制御します。エアギャップ環境では `false` に設定することを推奨します。

| フィールド | 型 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `allowAnonymousTelemetry` | 真偽値 | `true` | `false` に設定するとテレメトリ送信を無効化します |

```yaml
allowAnonymousTelemetry: false
```

!!! warning "監査要件がある場合"
    テレメトリを無効化しても、ネットワークレベルで外部通信を完全に遮断できているかは別途確認が必要です。パケットキャプチャやプロキシログを用いた検証手順については [第 4 章](../04-telemetry-airgap-verification.md) を参照してください。

---

## 完全な config.yaml 設定例

### 最小構成（Chat + Autocomplete のみ）

Ollama でモデルを 1 つずつ用意している場合の最小限の設定です。

```yaml
# ~/.continue/config.yaml  ——  最小構成

allowAnonymousTelemetry: false

models:
  - name: My Chat Model
    provider: ollama
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"

tabAutocompleteModel:
  name: My Autocomplete Model
  provider: ollama
  model: <your-autocomplete-model-name>
  apiBase: "http://<your-internal-llm-host>:11434"

contextProviders:
  - name: code
  - name: diff
  - name: terminal
  - name: problems
  - name: folder
  - name: file

slashCommands:
  - name: edit
    description: "選択範囲のコードを編集する"
  - name: comment
    description: "コードにコメントを追加する"
```

### フル構成（全機能有効テンプレート）

すべての主要機能を有効にした完全テンプレートです。プレースホルダを実際の値に置き換えて使用してください。

```yaml
# ~/.continue/config.yaml  ——  フル構成テンプレート

allowAnonymousTelemetry: false

# ─────────────────────────────────
# モデル設定（Chat / Edit / Agent）
# ─────────────────────────────────
models:
  - name: Large Model（高精度）
    provider: ollama
    model: <your-large-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"

  - name: Small Model（軽量）
    provider: ollama
    model: <your-small-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"

  # ── Embedding（@codebase / @docs 向け）──
  - name: Embeddings
    provider: ollama
    model: <your-embedding-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - embed

# ─────────────────────────────────
# Autocomplete 設定
# ─────────────────────────────────
tabAutocompleteModel:
  name: Autocomplete Model
  provider: ollama
  model: <your-autocomplete-model-name>
  apiBase: "http://<your-internal-llm-host>:11434"

tabAutocompleteOptions:
  maxPromptTokens: 1024
  debounceDelay: 350
  multilineCompletions: "auto"
  disableInFiles:
    - "*.md"

# ─────────────────────────────────
# コンテキストプロバイダ
# ─────────────────────────────────
contextProviders:
  - name: code
  - name: codebase
  - name: docs
  - name: diff
  - name: terminal
  - name: problems
  - name: folder
  - name: file
  - name: open
  - name: search
  - name: tree

# ─────────────────────────────────
# スラッシュコマンド
# ─────────────────────────────────
slashCommands:
  - name: edit
    description: "選択範囲のコードを編集する"
  - name: comment
    description: "コードにコメントを追加する"
  - name: share
    description: "会話をマークダウンとしてエクスポートする"
  - name: cmd
    description: "シェルコマンドを生成する"
  - name: commit
    description: "コミットメッセージを生成する"

# ─────────────────────────────────
# カスタムコマンド
# ─────────────────────────────────
customCommands:
  - name: review
    description: "コードをセキュリティ観点でレビューする"
    prompt: >-
      次のコードについて、セキュリティ上の問題点を日本語で指摘してください。
      問題がない場合は「問題なし」と答えてください。

      {{{ input }}}

# ─────────────────────────────────
# ルール（共通指示）
# ─────────────────────────────────
rules:
  - name: 日本語で回答する
    rule: "回答は必ず日本語で行ってください。"

# ─────────────────────────────────
# MCP サーバ
# ─────────────────────────────────
mcpServers:
  - name: filesystem
    command: node
    args:
      - "/opt/mcp-servers/filesystem/index.js"
      - "--root"
      - "/home/<your-username>/projects"
```

---

## 関連リンク

- [Continue 公式ドキュメント — Configuration Reference](https://docs.continue.dev/reference/config)
- [Continue 公式ドキュメント — Context Providers](https://docs.continue.dev/reference/context-providers)
- [Continue 公式ドキュメント — Model Providers](https://docs.continue.dev/reference/model-providers/ollama)
- [YAML 公式仕様 1.2.2](https://yaml.org/spec/1.2.2/)
- [第 5 章 config.yaml の基本構造](../05-config-yaml-basics.md)
- [第 7 章 Autocomplete（タブ補完）を使いこなす](../07-autocomplete.md)
- [第 10 章 社内モデルとコンテキスト管理](../10-internal-models-context.md)
- [第 11 章 カスタマイズで開発フローに最適化する](../11-customization.md)
- [第 12 章 MCP サーバの社内利用](../12-mcp-onprem.md)
