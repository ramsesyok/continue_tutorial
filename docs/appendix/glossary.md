---
title: "付録C 用語集"
type: appendix
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - glossary
  - reference
---

## この付録について

本付録は、このチュートリアル全体（第 1 章〜第 14 章および付録 A / B）で使用する主な専門用語を一か所にまとめたリファレンスです。各章を読み進める中で意味が分からない用語に出会ったときや、用語の定義を改めて確認したいときにご活用ください。

用語は次の 8 つのカテゴリに分類しています。

- [A. Continue 機能・画面](#a-continue-機能画面)
- [B. VS Code・拡張機能](#b-vs-code拡張機能)
- [C. LLM・AI 全般](#c-llmai-全般)
- [D. config.yaml・設定](#d-configyaml設定)
- [E. コンテキスト・インデックス](#e-コンテキストインデックス)
- [F. MCP（Model Context Protocol）](#f-mcpmodel-context-protocol)
- [G. エアギャップ・オンプレミス・セキュリティ](#g-エアギャップオンプレミスセキュリティ)
- [H. ネットワーク・ローカル LLM サーバ](#h-ネットワークローカル-llm-サーバ)

---

## 凡例の読み方

各エントリは次の書式で記述しています。

> **用語名**（読み） — _英語: English Term_
>
> 定義文。初出 → [第 N 章](../NN-slug.md)

「初出」はこのチュートリアルで初めて解説している章を示します。リンクをたどると該当箇所に移動できます。

---

## 用語一覧

### A. Continue 機能・画面

**Autocomplete（オートコンプリート）** — _英語: Autocomplete_

コードを入力している最中に、続きのコードを自動的に候補として表示する機能です。Tab キーを押すことで候補を確定でき、「タブ補完」とも呼ばれます。Chat モデルとは別の専用モデル（`tabAutocompleteModel`）を使うことが多く、低レイテンシが特に重視されます。初出 → [第 7 章](../07-autocomplete.md)

---

**Agent（エージェント）** — _英語: Agent_

ファイルの読み書き・コマンド実行などのツールを順番に呼び出しながら、複数ステップの作業を自律的に進める Continue の機能です。「思考 → ツール呼び出し → 観測 → 判断」というループを繰り返してゴールに近づきます。各操作はユーザーの承認を得てから実行されます（ツール承認モデルを参照）。初出 → [第 9 章](../09-agent.md)

---

**Chat（チャット）** — _英語: Chat_

AI に質問したり、コードの説明を求めたりするための対話機能です。ファイルの直接操作は行わず、テキストの応答を返します。`@file` や `@codebase` などのコンテキストプロバイダと組み合わせることで、コードや社内ドキュメントに基づいた回答を得られます。初出 → [第 6 章](../06-chat-basics.md)

---

**Edit（エディット）** — _英語: Edit_

VS Code で選択したコードを、ユーザーの指示に従って書き換える機能です。変更は差分（diff）形式で提示され、Accept または Reject を選んで確定します。Agent とは異なり、1 回の指示で 1 か所だけを書き換えるのが基本です。初出 → [第 8 章](../08-edit.md)

---

**Inline Edit（インライン Edit）** — _英語: Inline Edit_

エディタ上で選択範囲を指定し、その場で Edit を起動してコードを書き換える操作方法です。ショートカット（`Ctrl+I` / `Cmd+I`）またはコマンドパレットから呼び出します。初出 → [第 8 章](../08-edit.md)

---

**Accept / Reject（承認・却下）** — _英語: Accept / Reject_

Edit や Agent のツール呼び出しが提示した変更案に対して、ユーザーが確定（Accept）または破棄（Reject）を選択する操作です。Agent のツール呼び出しに対する Reject は変更を取り消すだけでなく、Agent に方針変更を伝えるきっかけにもなります。初出 → [第 8 章](../08-edit.md)

---

**ツール承認モデル** — _英語: Tool Approval Model_

Agent がファイル操作やコマンド実行を行う際に、ユーザーがそれを承認・拒否できる仕組みです。「自動承認（Auto）」「都度承認（Ask）」「拒否（Deny）」の 3 段階があり、ツールごとに個別に設定できます。エアギャップ環境ではファイル編集・コマンド実行を「都度承認」にすることが推奨されます。初出 → [第 9 章](../09-agent.md)

---

**Tool Calling（ツール呼び出し）** — _英語: Tool Calling / Function Calling_

LLM がテキストを生成するのではなく、定義済みのツール（関数）を選択して引数を JSON で組み立て、呼び出す能力のことです。Agent モードではこの機能を使って各操作を実行します。LLM が Tool Calling に対応していない場合、Agent は正常に動作しません。初出 → [第 9 章](../09-agent.md)

---

### B. VS Code・拡張機能

**拡張機能（かくちょうきのう）** — _英語: Extension_

VS Code に追加インストールすることで機能を拡張するプラグインです。Continue も VS Code の拡張機能として提供されています。エアギャップ環境では Marketplace に接続できないため、VSIX ファイルを使ったオフラインインストールを行います。初出 → [第 3 章](../03-offline-install.md)

---

**VSIX（ブイエスアイエックス）** — _英語: VSIX_

VS Code 拡張機能のインストールパッケージファイルです。拡張子は `.vsix` です。エアギャップ環境では VSIX ファイルを社内で配布し、VS Code の「拡張機能」ビューから「VSIX からインストール」する手順でセットアップします。初出 → [第 3 章](../03-offline-install.md)

---

**コマンドパレット** — _英語: Command Palette_

VS Code のあらゆるコマンドをキーワードで検索して実行できる入力欄です。`Ctrl+Shift+P`（macOS では `Cmd+Shift+P`）で呼び出します。Continue の設定ファイルを開いたり、`@docs` のインデックスを再構築したりする際にも使います。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**アクティビティバー** — _英語: Activity Bar_

VS Code の最左端に縦に並んでいるアイコン群の領域です。エクスプローラー・検索・ソース管理・拡張機能などのアイコンが並び、Continue のアイコンもここに表示されます。クリックするとサイドパネルが切り替わります。初出 → [第 6 章](../06-chat-basics.md)

---

**ワークスペース** — _英語: Workspace_

VS Code で開いているフォルダ（またはフォルダの集合）のことです。Continue の `@codebase` はワークスペース内のファイルをインデックスの対象にします。プロジェクトのルートに `.continue/` フォルダを置くと、そのワークスペース固有の設定として機能します。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**Frontmatter（フロントマター）** — _英語: Frontmatter_

Markdown ファイルの先頭に `---` で区切って記述する YAML 形式のメタデータブロックです。本チュートリアルでは `title`・`type`・`tags` の 3 項目を必須としています。MkDocs はこの情報をもとにページタイトルやナビゲーションを構築します。初出 → `AGENT_INSTRUCTIONS.md`

---

**diff（ディフ）** — _英語: diff_

2 つのテキストの差分を表す形式、またはその差分自体のことです。変更前の行頭に `-`、変更後の行頭に `+` を付けて表示するのが一般的です。Edit や Agent のファイル編集ツールは変更内容を diff 形式で示し、ユーザーが内容を確認してから Accept を選べます。初出 → [第 8 章](../08-edit.md)

---

### C. LLM・AI 全般

**LLM（エルエルエム）** — _英語: LLM（Large Language Model）_

大量のテキストデータを学習し、自然言語での会話・コード生成・要約などを行う AI モデルの総称です。「大規模言語モデル」とも呼ばれます。Continue の Chat・Autocomplete・Edit・Agent 機能はすべて LLM を利用しています。本チュートリアルでは社内ネットワーク内で稼働するローカル LLM を使用します。初出 → [第 1 章](../01-introduction.md)

---

**ハルシネーション（幻覚）** — _英語: Hallucination_

AI がもっともらしく見える誤った情報やコードを生成してしまう現象のことです。LLM の構造上、完全には排除できません。AI の回答は必ず人間の目で検証することが重要です。`@codebase` や `@docs` で正確なコンテキストを渡すことで、ハルシネーションを軽減できる場合があります。初出 → [第 1 章](../01-introduction.md)

---

**トークン（とーくん）** — _英語: Token_

LLM がテキストを処理する際の最小単位です。英語では単語やサブワードが 1 トークンに相当し、日本語では数文字から数十文字が 1 トークンに相当します。LLM が一度に処理できるトークン数の上限をコンテキストウィンドウと呼びます。初出 → [第 6 章](../06-chat-basics.md)

---

**コンテキストウィンドウ** — _英語: Context Window_

LLM が 1 回の推論で参照できるテキストの最大トークン数です。コンテキストウィンドウを超えた情報はモデルに渡らず、「忘れた」ような挙動になります。コンテキストウィンドウの大きいモデルほど長い会話や大きなファイルを扱えますが、一般的に推論が重くなります。初出 → [第 6 章](../06-chat-basics.md)

---

**プロンプト（ぷろんぷと）** — _英語: Prompt_

LLM に入力するテキスト（指示・質問・コードなど）の総称です。Continue では、ユーザーのメッセージに加えて `@file` や `@codebase` の取得内容が自動的にプロンプトに組み込まれます。適切なプロンプトを書くことで LLM の回答精度を高められます。初出 → [第 6 章](../06-chat-basics.md)

---

**推論（すいろん）** — _英語: Inference_

LLM が入力（プロンプト）を受け取り、出力（テキスト）を生成する処理のことです。Chat の応答生成・Autocomplete の候補生成・Edit の差分生成などはすべて推論です。GPU を用いた並列計算で処理されるため、モデルサイズが大きいほど推論に時間がかかります。初出 → [第 7 章](../07-autocomplete.md)

---

**Embedding（埋め込み）** — _英語: Embedding_

テキストを高次元のベクトル（数値の配列）に変換する処理、またはその変換結果のことです。Embedding モデルはテキストの意味を数値空間上の点として表現し、意味的に近いテキスト同士が近い位置に来るよう設計されています。`@codebase` や `@docs` の検索機能の基盤技術です。初出 → [第 10 章](../10-internal-models-context.md)

---

**FIM（フィルインザミドル）** — _英語: FIM（Fill-in-the-Middle）_

コードの「前」と「後」の両方のコンテキストを参照して、中間部分の補完を生成する LLM の推論手法です。Autocomplete（タブ補完）ではカーソルの前後のコードを参照するため、FIM に対応したモデルが高品質な補完を生成します。初出 → [第 7 章](../07-autocomplete.md)

---

**量子化（りょうしか）** — _英語: Quantization_

モデルの重みパラメータを低精度の数値型（例: INT4、INT8）に変換することで、モデルのサイズと VRAM 消費量を削減する技術です。精度はわずかに低下しますが、エアギャップ環境の限られた GPU リソース上でも大型モデルを動かせるようになります。`Q4` は INT4 量子化を指す略称としてよく使われます。初出 → [第 10 章](../10-internal-models-context.md)

---

**VRAM（ブイラム）** — _英語: VRAM（Video RAM）_

GPU に搭載されているメモリのことです。LLM の推論では、モデルの重みとその計算中間データをすべて VRAM に展開するため、大型モデルほど多くの VRAM が必要です。量子化によって消費量を削減できますが、複数のモデルを同時にホストする場合は合計 VRAM 容量に注意が必要です。初出 → [第 10 章](../10-internal-models-context.md)

---

### D. config.yaml・設定

**config.yaml（コンフィグヤムル）** — _英語: config.yaml_

Continue の動作を制御する設定ファイルです。接続する LLM のエンドポイント・モデル名・コンテキストプロバイダ・スラッシュコマンドなど、ほぼすべての設定をこのファイルに記述します。デフォルトのパスは `~/.continue/config.yaml`（Linux・macOS）または `%USERPROFILE%\.continue\config.yaml`（Windows）です。保存すると設定が即座にリロードされます。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**YAML（ヤムル）** — _英語: YAML（YAML Ain't Markup Language）_

人間が読み書きしやすいことを目的に設計されたデータシリアライズフォーマットです。`config.yaml` はこの形式で記述します。半角スペースによるインデントが階層構造を表し、タブ文字は使用できません。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**models キー（もでるずきー）** — _英語: `models` key_

`config.yaml` のトップレベルキーで、Chat・Edit・Apply・Embedding など（Autocomplete を除く）のロールで使用するモデルをリスト形式で登録します。各エントリに `name`・`provider`・`model`・`api_base`・`roles` などを指定します。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**tabAutocompleteModel（タブオートコンプリートモデル）** — _英語: `tabAutocompleteModel`_

`config.yaml` で Autocomplete 専用モデルを指定するキーです。`models` リストとは別に 1 つだけ記述します。低レイテンシが求められるため、小型のコード補完専用モデルを選ぶことが推奨されます。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**allowAnonymousTelemetry（あろうあのにますてれめとりー）** — _英語: `allowAnonymousTelemetry`_

Continue のテレメトリ（使用状況の自動送信）を有効・無効にする `config.yaml` のキーです。`false` に設定することでテレメトリを無効化できます。エアギャップ環境では必ず `false` に設定し、第 4 章の手順で実際に外部通信が止まっていることを確認してください。初出 → [第 4 章](../04-telemetry-airgap-verification.md)

---

**contextProviders（コンテキストプロバイダーズ）** — _英語: `contextProviders`_

`config.yaml` で Chat のコンテキスト機能（`@file`・`@codebase`・`@docs` など）を有効化するキーです。リストに記述されたプロバイダのみが Chat で利用できます。カスタムコンテキストプロバイダも同じリストに追加します。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**slashCommands（スラッシュコマンズ）** — _英語: `slashCommands`_

Chat の入力欄で `/edit`・`/comment` などのスラッシュコマンドを有効化するための `config.yaml` のキーです。よく使う操作をショートカットとして登録できます。独自のコマンドを追加するには `customCommands` キーを使います。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**rules（ルールズ）** — _英語: `rules`_

`config.yaml` に記述する、LLM へのグローバルな追加指示です。社内のコーディング規約・命名規則・使用言語の指定など、チームで共通の前提をモデルに伝えるために使います。Chat・Edit・Agent のすべてのやり取りに適用されます。初出 → [第 11 章](../11-customization.md)

---

**roles（ロールズ）** — _英語: `roles`_

`config.yaml` の `models` エントリに付与する、そのモデルの用途を示すリストです。`chat`・`edit`・`apply`・`embed`・`rerank` などの値が使えます。1 つのモデルに複数のロールを指定することも可能です。ロールを明示することで、Continue は用途ごとに適切なモデルを選択します。初出 → [第 10 章](../10-internal-models-context.md)

---

**provider（プロバイダ）** — _英語: `provider`_

`config.yaml` の `models` エントリで、モデルが動作しているサーバの種類を指定するキーです。`ollama`・`openai`（OpenAI 互換 API）・`lmstudio` などが主な選択肢です。vLLM は OpenAI 互換 API を提供するため `provider: openai` を使います。初出 → [第 5 章](../05-config-yaml-basics.md)

---

### E. コンテキスト・インデックス

**@file（アットファイル）** — _英語: `@file`_

Chat の入力欄でファイルを明示的に指定してコンテキストとして渡すためのコンテキストプロバイダです。`@file src/main.py` のように使います。`@codebase` が自動的に関連ファイルを選ぶのに対し、`@file` は参照先を自分で指定します。初出 → [第 6 章](../06-chat-basics.md)

---

**@codebase（アットコードベース）** — _英語: `@codebase`_

ワークスペース全体をローカル Embedding でインデックス化し、質問に関連するコードを自動的にコンテキストとして抽出するコンテキストプロバイダです。すべての処理はローカルで完結するため、エアギャップ環境でもそのまま利用できます。初出 → [第 6 章](../06-chat-basics.md) / [第 10 章](../10-internal-models-context.md)

---

**@docs（アットドキュメンツ）** — _英語: `@docs`_

社内ドキュメントサイトや Markdown ファイル群をインデックス化し、Chat のコンテキストとして参照できるコンテキストプロバイダです。エアギャップ環境では社内 HTTP サーバまたはローカルパスを参照先として指定します。外部 URL を参照先に記述するとエアギャップ前提が崩れるため、必ず社内・ローカルのソースのみを指定してください。初出 → [第 10 章](../10-internal-models-context.md)

---

**コンテキストプロバイダ** — _英語: Context Provider_

Chat に渡すコンテキスト情報の種類を定義するプラグイン機構です。`@file`・`@codebase`・`@docs`・`@terminal` などはそれぞれがコンテキストプロバイダの実装です。`config.yaml` の `contextProviders` キーで有効・無効を制御でき、HTTP プロバイダを使って自作することもできます。初出 → [第 6 章](../06-chat-basics.md) / [第 10 章](../10-internal-models-context.md)

---

**カスタムコンテキストプロバイダ** — _英語: Custom Context Provider_

標準のコンテキストプロバイダでは取り込めない社内独自の情報源を、`@<任意の名前>` で Chat から参照できるよう自作するプロバイダです。HTTP プロバイダを使って社内 LAN 上に小さなサーバを立て、社内 Issue・DB スキーマ・運用手順書などを返すように実装します。初出 → [第 10 章](../10-internal-models-context.md)

---

**Embedding モデル（エンベディングモデル）** — _英語: Embedding Model_

テキストを高次元ベクトルに変換することに特化した軽量なモデルです。Chat モデルのようにテキストを生成するのではなく、意味のベクトル表現を返します。`@codebase` と `@docs` の検索機能の土台となります。`config.yaml` では `roles: [embed]` を付与して登録します。初出 → [第 10 章](../10-internal-models-context.md)

---

**Reranker（リランカー）** — _英語: Reranker_

Embedding による類似度検索で得られた候補群を、質問との関連度に基づいて並べ替えるモデルです。Embedding 単独よりも精度の高いコンテキスト選択が可能になります。追加の VRAM とレイテンシが必要な任意機能であり、まず Embedding のみで運用してから、ノイズが気になった段階で追加するアプローチが現実的です。初出 → [第 10 章](../10-internal-models-context.md)

---

**チャンク分割（チャンクぶんかつ）** — _英語: Chunking_

`@codebase` や `@docs` のインデックス構築時に、ソースファイルやドキュメントを Embedding に適した大きさ（典型的には数百トークン程度）のブロックに分割する処理です。コードでは関数やクラスの境界が、ドキュメントでは見出しが分割の目安になります。初出 → [第 10 章](../10-internal-models-context.md)

---

**インデックス** — _英語: Index_

`@codebase` や `@docs` が構築するローカルの検索データベースです。各チャンクの Embedding ベクトル・ファイルパス・行番号などが格納されます。`~/.continue/index/` 配下に保存されます。Embedding モデルを変更したり `.continueignore` を大幅に変更したりした場合は、再構築が必要です。初出 → [第 10 章](../10-internal-models-context.md)

---

**.continueignore（ドットコンティニューイグノア）** — _英語: `.continueignore`_

`@codebase` インデックスの対象から除外するパスやパターンを指定するファイルです。`.gitignore` と同じ書式で記述します。自動生成ファイル・依存ライブラリのソース・バイナリファイルなど、検索に不要なものを除外することで、インデックスの品質と構築速度を改善できます。初出 → [第 10 章](../10-internal-models-context.md)

---

### F. MCP（Model Context Protocol）

**MCP（エムシーピー）** — _英語: MCP（Model Context Protocol）_

LLM がファイル操作・データベース参照・コマンド実行といった外部ツールやデータソースと、標準化されたインターフェースでやり取りするためのプロトコルです。2024 年に Anthropic が策定・公開しました。Continue の Agent モードでの外部ツール連携に使用されます。初出 → [第 12 章](../12-mcp-onprem.md)

---

**MCP サーバ（エムシーピーサーバ）** — _英語: MCP Server_

MCP プロトコルを実装し、LLM からのツール呼び出し要求を受け付けて実際の処理（ファイル読み書き・DB クエリなど）を行うサーバプログラムです。Node.js または Python で実装されることが多いです。エアギャップ環境では外部通信が不要なローカル操作型のみを使用します。初出 → [第 12 章](../12-mcp-onprem.md)

---

**ツール（MCP のツール）** — _英語: Tool（in MCP context）_

MCP サーバが LLM に対して提供する操作の単位です。「ファイルを読む」「SQL をクエリする」「Git ログを取得する」などがツールの例です。LLM はツールの一覧と仕様を受け取った上で、必要なツールを選び引数を組み立てて呼び出します。初出 → [第 12 章](../12-mcp-onprem.md)

---

**mcpServers（エムシーピーサーバーズ）** — _英語: `mcpServers`_

`config.yaml` に MCP サーバの接続情報を登録するキーです。各エントリに `name`・`command`・`args` などを指定します。登録した MCP サーバのツールは Agent モードで自動的に利用可能になります。初出 → [第 12 章](../12-mcp-onprem.md)

---

**ローカル操作型 MCP サーバ** — _英語: Local MCP Server_

ファイルシステム・社内 Git・ローカルデータベースなど、社内ネットワーク内のリソースのみを操作する MCP サーバです。外部通信を必要とせず、エアギャップ環境で利用できます。外部 SaaS（GitHub.com・Slack など）に接続する外部 API 型は本構成では使用しません。初出 → [第 12 章](../12-mcp-onprem.md)

---

### G. エアギャップ・オンプレミス・セキュリティ

**エアギャップ（エアギャップ）** — _英語: Air-gap_

ネットワーク上で完全に隔離された環境のことです。インターネットや外部ネットワークと物理的・論理的に遮断されており、外部との通信が一切できません。機密情報を扱う組織のシステムに採用されることが多く、本チュートリアルはこの環境での Continue 利用を前提としています。初出 → [第 1 章](../01-introduction.md)

---

**オンプレミス（オンプレミス）** — _英語: On-premises_

サーバやソフトウェアをクラウドサービスに頼らず、自社施設内（社内ネットワーク内）に設置・運用する形態のことです。本チュートリアルでは、LLM サーバも社内ネットワーク内に置くオンプレミス構成を前提とします。初出 → [第 1 章](../01-introduction.md)

---

**テレメトリ（テレメトリ）** — _英語: Telemetry_

ソフトウェアが動作状況・利用傾向などのデータをネットワーク越しに自動送信する仕組みのことです。Continue はデフォルトで匿名の使用統計を送信する設定になっています。エアギャップ環境では `allowAnonymousTelemetry: false` を設定して無効化し、パケットキャプチャなどで実際に送信が止まっていることを確認します。初出 → [第 4 章](../04-telemetry-airgap-verification.md)

---

**パケットキャプチャ** — _英語: Packet Capture_

ネットワーク上を流れるパケット（データの断片）を記録・解析するツールや操作のことです。Wireshark などのツールを使い、外部サーバへの通信が発生していないことを確認するためのエビデンス収集手段として活用します。初出 → [第 4 章](../04-telemetry-airgap-verification.md)

---

**プロキシ（プロキシ）** — _英語: Proxy_

ネットワーク上でクライアントとサーバの間に介在し、通信の中継・記録・フィルタリングを行うサーバです。エアギャップ環境や閉域網では、プロキシのログを確認することで外部通信の有無を証明できます。VS Code 開発者ツールと組み合わせた通信遮断確認の手段としても使われます。初出 → [第 4 章](../04-telemetry-airgap-verification.md)

---

**監査ログ（カンサログ）** — _英語: Audit Log_

誰がいつ何をしたかを記録したログです。Continue の利用ログやパケットキャプチャの記録は、セキュリティ監査の際に「外部通信が発生していないこと」を証明するエビデンスとして提出できます。運用ポリシーに応じた保管期間と保管場所の設計が必要です。初出 → [第 4 章](../04-telemetry-airgap-verification.md) / [第 13 章](../13-security-operations.md)

---

**最小権限の原則（サイショウケンゲンのゲンソク）** — _英語: Principle of Least Privilege_

ユーザーやプログラムには、その作業に必要な最低限の権限だけを付与するというセキュリティ原則です。Agent のツール承認設定においても、コマンド実行のスコープを絞ることでリスクを低減できます。初出 → [第 13 章](../13-security-operations.md)

---

**社内ミラー（シャナイミラー）** — _英語: Internal Mirror_

インターネット上のパッケージリポジトリやソフトウェアを社内ネットワーク内に複製（ミラー）したサーバです。エアギャップ環境では Python パッケージ（pip）・Node.js パッケージ（npm）・VS Code 拡張機能（VSIX）などを社内ミラーから取得します。初出 → [第 3 章](../03-offline-install.md)

---

### H. ネットワーク・ローカル LLM サーバ

**Ollama（オラマ）** — _英語: Ollama_

コマンド 1 つでローカル LLM を起動・管理できるオープンソースツールです。`config.yaml` では `provider: ollama` として設定します。デフォルトポートは `11434` で、Chat・Autocomplete・Embedding など複数のモデルを同時にホストできます。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**vLLM（ブイエルエルエム）** — _英語: vLLM_

高スループットな LLM 推論エンジンです。OpenAI 互換 API を提供するため、`config.yaml` では `provider: openai` として設定します。大規模モデルをサーバ運用する場合に向きます。デフォルトポートは `8000` で、エンドポイントパスは `/v1` です。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**LM Studio（エルエムスタジオ）** — _英語: LM Studio_

GUI でローカル LLM を管理・実行できるツールです。OpenAI 互換サーバを内蔵しており、`config.yaml` では `provider: lmstudio` または `provider: openai` として設定します。デフォルトポートは `1234` です。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**エンドポイント（エンドポイント）** — _英語: Endpoint_

HTTP ベースの API を呼び出す際の URL のことです。`config.yaml` の `api_base`（または `apiBase`）キーに LLM サーバのエンドポイントを記述します。プロトコル・ホスト名・ポート番号・パス（`/v1` など）で構成されます。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**OpenAI 互換 API（オープンエーアイゴカンエーピーアイ）** — _英語: OpenAI-compatible API_

OpenAI が公開した API 仕様と互換性のある HTTP API のことです。vLLM・LM Studio など多くのローカル LLM サーバがこの仕様に準拠しており、`provider: openai` として同じ設定方法で接続できます。初出 → [第 5 章](../05-config-yaml-basics.md)

---

**ヘルスチェック（ヘルスチェック）** — _英語: Health Check_

LLM サーバが正常に稼働しているかを確認する疎通確認操作のことです。`curl` コマンドで LLM サーバのエンドポイントにリクエストを送り、応答が返ってくることで動作を確認します。Continue から応答がない場合のトラブルシューティングの第一ステップです。初出 → [第 5 章](../05-config-yaml-basics.md)

---

## 索引

本付録に収録した用語の索引です。日本語の見出しはおおむね五十音順、英数字から始まる見出しはアルファベット順で並べています。

### 日本語

| 用語 | カテゴリ |
| --- | --- |
| アクティビティバー | [B](#b-vs-code拡張機能) |
| エアギャップ | [G](#g-エアギャップオンプレミスセキュリティ) |
| エンドポイント | [H](#h-ネットワークローカル-llm-サーバ) |
| オープン AI 互換 API → OpenAI 互換 API | [H](#h-ネットワークローカル-llm-サーバ) |
| オラマ → Ollama | [H](#h-ネットワークローカル-llm-サーバ) |
| オンプレミス | [G](#g-エアギャップオンプレミスセキュリティ) |
| カスタムコンテキストプロバイダ | [E](#e-コンテキストインデックス) |
| 監査ログ | [G](#g-エアギャップオンプレミスセキュリティ) |
| コマンドパレット | [B](#b-vs-code拡張機能) |
| コンテキストウィンドウ | [C](#c-llmai-全般) |
| コンテキストプロバイダ | [E](#e-コンテキストインデックス) |
| 最小権限の原則 | [G](#g-エアギャップオンプレミスセキュリティ) |
| 社内ミラー | [G](#g-エアギャップオンプレミスセキュリティ) |
| チャンク分割 | [E](#e-コンテキストインデックス) |
| ツール（MCP） | [F](#f-mcpmodel-context-protocol) |
| ツール承認モデル | [A](#a-continue-機能画面) |
| テレメトリ | [G](#g-エアギャップオンプレミスセキュリティ) |
| トークン | [C](#c-llmai-全般) |
| 拡張機能 | [B](#b-vs-code拡張機能) |
| ハルシネーション | [C](#c-llmai-全般) |
| パケットキャプチャ | [G](#g-エアギャップオンプレミスセキュリティ) |
| プロキシ | [G](#g-エアギャップオンプレミスセキュリティ) |
| プロンプト | [C](#c-llmai-全般) |
| ヘルスチェック | [H](#h-ネットワークローカル-llm-サーバ) |
| ワークスペース | [B](#b-vs-code拡張機能) |
| リランカー → Reranker | [E](#e-コンテキストインデックス) |
| ローカル操作型 MCP サーバ | [F](#f-mcpmodel-context-protocol) |
| 量子化 | [C](#c-llmai-全般) |
| 推論 | [C](#c-llmai-全般) |

### 英数字

| 用語 | カテゴリ |
| --- | --- |
| `@codebase` | [E](#e-コンテキストインデックス) |
| `@docs` | [E](#e-コンテキストインデックス) |
| `@file` | [E](#e-コンテキストインデックス) |
| Accept / Reject | [A](#a-continue-機能画面) |
| Agent | [A](#a-continue-機能画面) |
| `allowAnonymousTelemetry` | [D](#d-configyaml設定) |
| Autocomplete | [A](#a-continue-機能画面) |
| Chat | [A](#a-continue-機能画面) |
| config.yaml | [D](#d-configyaml設定) |
| `.continueignore` | [E](#e-コンテキストインデックス) |
| `contextProviders` | [D](#d-configyaml設定) |
| diff | [B](#b-vs-code拡張機能) |
| Edit | [A](#a-continue-機能画面) |
| Embedding | [C](#c-llmai-全般) |
| Embedding モデル | [E](#e-コンテキストインデックス) |
| FIM | [C](#c-llmai-全般) |
| Frontmatter | [B](#b-vs-code拡張機能) |
| Inline Edit | [A](#a-continue-機能画面) |
| インデックス | [E](#e-コンテキストインデックス) |
| LLM | [C](#c-llmai-全般) |
| LM Studio | [H](#h-ネットワークローカル-llm-サーバ) |
| MCP | [F](#f-mcpmodel-context-protocol) |
| MCP サーバ | [F](#f-mcpmodel-context-protocol) |
| `mcpServers` | [F](#f-mcpmodel-context-protocol) |
| `models` キー | [D](#d-configyaml設定) |
| Ollama | [H](#h-ネットワークローカル-llm-サーバ) |
| OpenAI 互換 API | [H](#h-ネットワークローカル-llm-サーバ) |
| `provider` | [D](#d-configyaml設定) |
| Reranker | [E](#e-コンテキストインデックス) |
| `roles` | [D](#d-configyaml設定) |
| `rules` | [D](#d-configyaml設定) |
| `slashCommands` | [D](#d-configyaml設定) |
| `tabAutocompleteModel` | [D](#d-configyaml設定) |
| Tool Calling | [A](#a-continue-機能画面) |
| VRAM | [C](#c-llmai-全般) |
| VSIX | [B](#b-vs-code拡張機能) |
| vLLM | [H](#h-ネットワークローカル-llm-サーバ) |
| YAML | [D](#d-configyaml設定) |

---

## 参考リンク

- [Continue 公式ドキュメント](https://docs.continue.dev/)（用語の最新定義は公式サイトを参照してください）
- [MkDocs 公式サイト](https://www.mkdocs.org/)
- [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/)
- [Foam（VS Code 用ナレッジベース拡張）](https://github.com/foambubble/foam)
