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

本付録は、Continue の設定ファイル `config.yaml` に記述できる全キー・全オプションを網羅した辞書的リファレンスです。チュートリアル形式で最小構成を学ぶ [第5章](../05-config-yaml-basics.md) の補足として、手元に置いて引くことを想定しています。

各節には「詳細は第 N 章」という相互参照を記載しており、設定の背景や操作手順は対応する章を参照してください。<glossary:エアギャップ>環境では使用できない機能には「**本構成では使用しません**」と明示しています。

---

## トップレベルキー早見表

`config.yaml` に記述できる全トップレベルキーを以下に示します。

| キー | 型 | 必須 | 概要 | 詳細 |
| --- | --- | --- | --- | --- |
| `name` | 文字列 | ✓ | 設定の名前 | — |
| `version` | 文字列 | ✓ | 設定のバージョン（例: `1.0.0`） | — |
| `schema` | 文字列 | ✓ | スキーマバージョン（`v1` 固定） | — |
| `models` | リスト | ✓ | <glossary:Chat>・<glossary:Edit>・<glossary:Agent>・<glossary:Autocomplete>・<glossary:Embedding> で使うモデルの一覧（`roles:` フィールドで用途を指定） | [models](#models) / 第5・6・7・10章 |
| `context` | リスト | — | `@file` / `@codebase` などの参照機能 | [context](#context) / 第6・10章 |
| `rules` | リスト | — | モデルへの共通指示（コーディング規約など） | [rules](#rules) / 第11章 |
| `prompts` | リスト | — | ユーザー定義のカスタム<glossary:プロンプト>（旧 `customCommands`） | [prompts](#prompts) / 第11章 |
| `mcpServers` | リスト | — | <glossary:MCPサーバ>の接続設定 | [mcpServers](#mcpservers) / 第12章 |

!!! note "テレメトリ設定について"
    旧バージョンの `allowAnonymousTelemetry:` キーは現在の `config.yaml` では使用しません。<glossary:テレメトリ>の有効・無効は VS Code の設定（**継続 › テレメトリ**）または `settings.json` の `"continue.telemetryEnabled"` で制御してください（→ 詳細は [第4章](../04-telemetry-airgap-verification.md)）。

---

## models

Chat・Edit・Agent・Autocomplete・Embedding で使うモデルをすべてこのリストに登録します。各エントリの `roles:` フィールドで用途を指定します。

### 共通フィールド一覧

| フィールド | 型 | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- | --- |
| `name` | 文字列 | ✓ | — | VS Code の UI に表示されるモデルの表示名。任意の文字列を指定できます |
| `provider` | 文字列 | ✓ | — | 接続するバックエンドの種別。`ollama` / `openai` / `lmstudio` など |
| `model` | 文字列 | ✓ | — | バックエンドに登録されているモデル名。プロバイダ側の識別子と一致させます |
| `apiBase` | 文字列 | — | プロバイダ依存 | <glossary:LLM> サーバの<glossary:エンドポイント> URL。プロバイダのデフォルトから変更する場合に指定します |
| `apiKey` | 文字列 | — | — | 認証<glossary:トークン>。社内サーバで認証が不要でも、プロバイダによっては任意の文字列が必要な場合があります |
| `roles` | リスト | — | プロバイダ依存 | このモデルが担う役割。`chat`・`edit`・`autocomplete`・`embed`・`rerank` を指定できます |
| `autocompleteOptions` | オブジェクト | — | — | `roles` に `autocomplete` を含む場合の補完チューニングオプション |
| `contextLength` | 整数 | — | プロバイダ依存 | モデルが受け付ける最大コンテキスト長（トークン数）。指定しない場合はプロバイダのデフォルト値が適用されます |
| `promptTemplates` | オブジェクト | — | — | `chat` / `edit` などのプロンプトテンプレートを上書きする場合に指定します |

### roles フィールド

`roles:` に指定できる値は以下の通りです。

| 値 | 用途 |
| --- | --- |
| `chat` | チャット・質問応答（メインの会話） |
| `edit` | インラインエディット（コードの書き換え） |
| `autocomplete` | タブ補完（Autocomplete） |
| `embed` | ベクトル Embedding（`@codebase` / `@docs` 向け） |
| `rerank` | 検索結果のリランキング |

### プロバイダ別設定

#### ollama

<glossary:Ollama> のデフォルトポートは `11434`、`apiBase` を省略すると `http://localhost:11434` が使われます。

```yaml
models:
  - name: Llama 3（Ollama）
    provider: ollama
    model: <your-model-name>         # 例: llama3、qwen2.5:7b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit
```

| 固有フィールド | 型 | 説明 |
| --- | --- | --- |
| `apiBase` | 文字列 | Ollama サーバの URL。別ホストで動かす場合に指定します |

#### openai（vLLM / LM Studio OpenAI 互換モード含む）

<glossary:vLLM> や <glossary:LMStudio> の <glossary:OpenAI互換API> にも `provider: openai` を使います。`apiBase` でエンドポイントを社内サーバに向けてください。

```yaml
models:
  - name: Qwen 2.5 72B（vLLM）
    provider: openai
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"                 # 認証不要でもフィールドが必要な場合があります
    roles:
      - chat
      - edit
```

| 固有フィールド | 型 | 説明 |
| --- | --- | --- |
| `apiBase` | 文字列 | OpenAI 互換サーバの URL。末尾 `/v1` まで含めて指定します |
| `apiKey` | 文字列 | API キー。認証不要な構成でも空白以外の文字列が必要な場合があります |

!!! warning "vLLM の apiKey"
    vLLM を認証なしで運用している場合でも、`apiKey` フィールドに `"dummy"` など任意の文字列を設定しないとエラーになる構成があります。社内サーバの設定に合わせて調整してください。

#### lmstudio

LM Studio 専用プロバイダです。デフォルトポート `1234` が自動で適用されるため、同一マシンで動かす場合は `apiBase` を省略できます。

```yaml
models:
  - name: Llama 3（LM Studio）
    provider: lmstudio
    model: <your-model-name>
    # apiBase を省略すると http://localhost:1234 が使われます
    # 別ホストで動かす場合は明示してください
    apiBase: "http://<your-internal-llm-host>:1234"
    roles:
      - chat
      - edit
```

### 複数モデルの登録と切り替え

`models` リストに複数エントリを並べると、VS Code の Continue パネルのドロップダウンで切り替えられます。用途（高精度タスク・軽量タスク）に応じてモデルを使い分けるのが実用的です。

```yaml
models:
  - name: Llama 3 70B（高精度）
    provider: ollama
    model: llama3:70b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit

  - name: Qwen 2.5 7B（軽量）
    provider: ollama
    model: qwen2.5:7b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit
```

---

## Autocomplete モデルの設定（roles: [autocomplete]）

Autocomplete（タブ補完）専用モデルは、`models` リストにエントリを追加し、`roles` フィールドに `autocomplete` を指定して登録します。補完チューニングのオプションは同じエントリ内の `autocompleteOptions:` に記述します（→ 詳細は [第7章](../07-autocomplete.md)）。

### autocompleteOptions フィールド一覧

| フィールド | 型 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `useCopyBuffer` | 真偽値 | `false` | コピーバッファの内容をコンテキストに含めるかどうか |
| `useFileSuffix` | 真偽値 | `true` | カーソル後方のコードをコンテキストに含めるかどうか（<glossary:FIM> 対応モデル向け） |
| `maxPromptTokens` | 整数 | `1024` | プロンプトに含める最大トークン数。増やすと精度が上がる場合がありますが、レイテンシも増加します |
| `prefixPercentage` | 数値 | `0.85` | プロンプトのうちカーソル前方に割り当てる割合（0〜1） |
| `maxSuffixPercentage` | 数値 | `0.25` | プロンプトのうちカーソル後方に割り当てる最大割合（0〜1） |
| `debounceDelay` | 整数（ms） | `350` | キー入力から補完リクエスト送信までの待機時間。低速なモデルでは大きくするとリクエストが減ります |
| `multilineCompletions` | 文字列 | `"auto"` | 複数行補完の動作。`"always"` / `"never"` / `"auto"` のいずれか |
| `disableInFiles` | リスト | `[]` | 補完を無効にするファイルのグロブパターン一覧（例: `["*.md", "*.txt"]`） |

```yaml
models:
  - name: 補完モデル（Qwen 2.5 Coder 1.5B）
    provider: ollama
    model: qwen2.5-coder:1.5b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - autocomplete
    autocompleteOptions:
      maxPromptTokens: 512       # 低速環境ではトークン数を減らしてレイテンシを改善
      debounceDelay: 500
      multilineCompletions: "auto"
      disableInFiles:
        - "*.md"
        - "*.txt"
```

!!! tip "レイテンシが気になる場合"
    `debounceDelay` を増やす・`maxPromptTokens` を減らす・`multilineCompletions: "never"` にする、の 3 つがレイテンシ改善の基本アプローチです。詳細は [第7章](../07-autocomplete.md) を参照してください。

---

## context

`@file`・`@codebase` などの参照機能（<glossary:コンテキストプロバイダ>）を有効化するリストです（→ 詳細は [第6章](../06-chat-basics.md) / [第10章](../10-internal-models-context.md)）。各エントリには `name:` ではなく `provider:` キーを使用します。

### 組み込みコンテキストプロバイダ一覧

| <glossary:provider> | 呼び出し方 | 概要 | エアギャップ対応 |
| --- | --- | --- | --- |
| `code` | `@code` | 関数・クラス等のシンボルを検索して参照 | ✓ |
| `codebase` | `@codebase` | コードベース全体を<glossary:インデックス>して参照 | ✓（ローカル Embedding 必須）|
| `docs` | `@docs` | ドキュメントサイトをインデックスして参照 | ✓（ローカルで事前取得が必要）|
| `diff` | `@diff` | 現在の Git <glossary:diff> を参照 | ✓ |
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

コードベース全体をベクトル検索で参照します。`models` リストにローカル <glossary:Embeddingモデル>（`roles: [embed]`）の設定が必須です。

```yaml
context:
  - provider: codebase
```

設定フィールドはなく、`provider: codebase` を追加するだけで有効になります。インデックス構築時間はリポジトリの規模に依存します。

#### docs

ドキュメントサイトをクローリングしてインデックス化します。エアギャップ環境では、外部 URL はアクセスできないため、**社内で配信しているドキュメントサイトの URL のみ**指定してください。

```yaml
context:
  - provider: docs
```

初回利用時に VS Code の Continue パネルから「Add Doc」でサイト URL を登録します。

!!! warning "エアギャップ環境での制限"
    外部のドキュメントサイト（例: 公式ドキュメントサイト）は、エアギャップ環境からアクセスできないため `@docs` でインデックス化できません。社内イントラネットで配信しているサイトのみ対象となります。

#### folder

```yaml
context:
  - provider: folder
```

チャット入力で `@folder` と入力すると、参照したいフォルダをインタラクティブに選択できます。

#### file

```yaml
context:
  - provider: file
```

`@file` と入力するとファイル名のオートコンプリートが起動します。

#### url

```yaml
context:
  - provider: url
```

`@url` に続けて URL を指定すると、そのページの内容が取得されてコンテキストに含まれます。エアギャップ環境では、ファイアウォール等で許可された社内 URL にのみ使用してください。

---

## prompts

`/` コマンドからプロンプト本文を直接 `config.yaml` に記述して独自コマンドを定義します（旧称: `customCommands`、→ 詳細は [第11章](../11-customization.md)）。

### フィールド一覧

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `name` | 文字列 | ✓ | コマンド名。`/` に続けて入力する識別子 |
| `description` | 文字列 | ✓ | コマンド一覧に表示される説明文 |
| `prompt` | 文字列 | ✓ | モデルに送信するプロンプト本文。`{{{ input }}}` で追加の入力を受け取れます |

```yaml
prompts:
  - name: review
    description: "コードをセキュリティ観点でレビューする"
    prompt: >-
      次のコードについて、セキュリティ上の問題点を日本語で指摘してください。
      問題がない場合は「問題なし」と答えてください。

      {{{ input }}}
```

!!! note "旧 slashCommands について"
    旧バージョンの `slashCommands:` キーは現在の `config.yaml` では使用しません（`/edit`・`/comment` などの組み込みコマンドは引き続き UI から使用できます）。カスタムコマンドを定義する場合は `prompts:` を使用してください。

---

## rules

モデルへの共通指示を記述するリストです。すべての会話・編集・エージェント操作に自動的に付与されます。社内コーディング規約や回答フォーマットの指定に活用してください（→ 詳細は [第11章](../11-customization.md)）。

### フォーマット

`rules:` には文字列、またはファイル参照（`uses: file://...`）を指定します。

```yaml
rules:
  - "回答は必ず日本語で行ってください。コードのコメントも日本語で記述してください。"
  - uses: "file://./.continue/rules/python-style.md"
```

ルールをファイルに分離して管理する場合は、プロジェクトフォルダ内の `.continue/rules/` ディレクトリに `.md` ファイルを置き、`uses: file://...` で参照します。

!!! tip "ルールファイルの管理"
    `.continue/rules/` ディレクトリ（プロジェクトルートのフォルダ内）にルールファイルを置くと、プロジェクトメンバー間で共有しやすくなります。チームのコーディング規約をファイルとして管理することで、バージョン管理の対象にもなります。

---

## Embedding モデルの設定（roles: [embed]）

`@codebase` や `@docs` 機能でコードベースをベクトル検索するために使うローカル Embedding モデルは、`models` リストにエントリを追加し、`roles` フィールドに `embed` を指定して登録します（→ 詳細は [第10章](../10-internal-models-context.md)）。

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
    apiKey: "dummy"
    roles:
      - embed
```

!!! note "本構成では使用しません"
    クラウドの OpenAI Embeddings API（`text-embedding-ada-002` など）はエアギャップ環境では使用できません。必ずローカルで動作する Embedding モデルを指定してください。

---

## mcpServers

<glossary:MCP>（Model Context Protocol）サーバへの接続を設定するリストです（→ 詳細は [第12章](../12-mcp-onprem.md)）。ローカルまたは社内で動作する MCP サーバのみ接続してください。

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

## 完全な config.yaml 設定例

### 最小構成（Chat + Autocomplete のみ）

Ollama でモデルを 1 つずつ用意している場合の最小限の設定です。

```yaml
# .continue/config.yaml  ——  最小構成

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
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: file
```

### フル構成（全機能有効テンプレート）

すべての主要機能を有効にした完全テンプレートです。プレースホルダを実際の値に置き換えて使用してください。

```yaml
# .continue/config.yaml  ——  フル構成テンプレート

name: my-continue-config
version: 1.0.0
schema: v1

# ─────────────────────────────────
# モデル設定（Chat / Edit / Agent / Autocomplete / Embedding）
# ─────────────────────────────────
models:
  - name: Large Model（高精度）
    provider: ollama
    model: <your-large-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit

  - name: Small Model（軽量）
    provider: ollama
    model: <your-small-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - chat
      - edit

  # ── Autocomplete ──
  - name: Autocomplete Model
    provider: ollama
    model: <your-autocomplete-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - autocomplete
    autocompleteOptions:
      maxPromptTokens: 1024
      debounceDelay: 350
      multilineCompletions: "auto"
      disableInFiles:
        - "*.md"

  # ── Embedding（@codebase / @docs 向け）──
  - name: Embeddings
    provider: ollama
    model: <your-embedding-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - embed

# ─────────────────────────────────
# コンテキストプロバイダ
# ─────────────────────────────────
context:
  - provider: code
  - provider: codebase
  - provider: docs
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: file
  - provider: open
  - provider: search
  - provider: tree

# ─────────────────────────────────
# カスタムプロンプト（旧 customCommands）
# ─────────────────────────────────
prompts:
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
  - "回答は必ず日本語で行ってください。"

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
- [第5章 config.yaml の基本構造](../05-config-yaml-basics.md)
- [第7章 Autocomplete（タブ補完）を使いこなす](../07-autocomplete.md)
- [第10章 社内モデルとコンテキスト管理](../10-internal-models-context.md)
- [第11章 カスタマイズで開発フローに最適化する](../11-customization.md)
- [第12章 MCP サーバの社内利用](../12-mcp-onprem.md)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    