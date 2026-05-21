---
title: "第10章 社内モデルとコンテキスト管理"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - context
  - embedding
  - indexing
---

## この章で学ぶこと

- 社内向けに複数の <glossary:LLM> をロール別へ割り当てる、実運用視点での設計指針
- `@codebase` と `@docs` の仕組みと、<glossary:エアギャップ>環境での<glossary:インデックス>運用
- ローカル <glossary:Embeddingモデル>・<glossary:Reranker> モデルを `config.yaml` に組み込む方法
- 用途に応じた `@codebase` と `@docs` の使い分け
- <glossary:カスタムコンテキストプロバイダ>を作って社内独自の情報源を取り込む手順（ハンズオン）

---

## 「社内モデル」と「コンテキスト管理」の関係

ここまでの章では、<glossary:Chat>（[第 6 章](06-chat-basics.md)）、<glossary:Autocomplete>（[第 7 章](07-autocomplete.md)）、<glossary:Edit>（[第 8 章](08-edit.md)）、<glossary:Agent>（[第 9 章](09-agent.md)）という、Continue の機能を 1 つずつ扱ってきました。これらに共通するのは、LLM への入力が **「どのモデルを使うか」** と **「どんなコンテキスト（情報）をモデルに渡すか」** の 2 つの軸で決まる、という事実です。

| 軸 | 何を決めるか | 主に扱う章 |
| --- | --- | --- |
| モデルの選定 | 速度・精度・<glossary:VRAM> のバランス、用途別の使い分け | 第 5 章・本章 10.2 |
| コンテキストの設計 | コードベース・社内文書・現在の編集状態を、どこまでモデルに見せるか | 第 6 章・本章 10.3〜10.7 |

この 2 つは独立しているように見えて、実運用では切っても切り離せません。高性能なモデルを用意しても、コンテキストとして渡される情報が貧しければ「自社の事情を知らない汎用回答」しか返ってきません。逆に豊富なコンテキストを渡せても、ロール別にモデルが適切に振り分けられていないと、補完が遅すぎたり、Chat の応答品質が足りなかったりします。

### エアギャップ環境ではコンテキスト管理が特に効く

クラウド型の AI コーディング支援サービスでは、公開リポジトリや一般公開ドキュメントの巨大な学習データに頼って「それなりの回答」を返せます。しかし<glossary:オンプレミス>・エアギャップ環境では、モデルが学習しているのは公開データだけです。社内独自のフレームワーク、社内 API、社内ライブラリの命名規則、社内のコーディング規約は、**コンテキストとして明示的に渡さない限り、モデルには見えません**。

つまり、エアギャップ環境で Continue を活用する鍵は「社内特有の情報をいかに効率よくモデルに渡すか」にあります。本章ではそのための仕組み（`@codebase`、`@docs`、ローカル <glossary:Embedding>、カスタム<glossary:コンテキストプロバイダ>）を順に扱います。

!!! note "本章の位置づけ"
    第 5 章で `config.yaml` の最小構成を扱い、第 6 章で `@file` や `@codebase` の基本的な使い方を紹介しました。本章では、これらを **実運用に耐える形に拡張する**ことに焦点を当てます。最小構成からの「次の一歩」が本章の到達点です。

---

## ロール別モデル構成を運用視点で設計する

第 5 章では、Chat 用の `models` と Autocomplete 用の `tabAutocompleteModel` を分けて指定する最小構成を扱いました。実運用では、ここからもう一段踏み込み、**用途（ロール）ごとに最適なモデルを割り当てる**設計が必要になります。

### ロールごとの要件を整理する

Continue が扱うロールと、それぞれに求められる典型的な要件を以下にまとめます。

| ロール | 主な役割 | レイテンシ要件 | 品質要件 | 適したモデルサイズの目安 |
| --- | --- | --- | --- | --- |
| Chat | 対話・調査・設計議論 | 数秒許容 | 高い | 中〜大型（7B〜70B クラス） |
| Edit | 選択範囲の書き換え・リファクタリング | 数秒許容 | 高い | 中〜大型（7B〜70B クラス） |
| Apply | Edit/Chat の出力を実ファイルへ適用 | 速い方が良い | 中〜高 | 中型（7B〜14B） |
| Autocomplete | 入力中のタブ補完 | 数百ミリ秒以内 | 補完特化 | 小型（1B〜3B、Coder 系） |
| Embed | `@codebase` `@docs` のベクトル検索 | バッチ処理向け | 検索精度重視 | 専用 Embedding モデル |
| Rerank | 検索結果の再並べ替え（任意） | 数百ミリ秒以内 | 関連度判定 | 専用 Reranker モデル |

ここで重要なのは、**1 つの巨大モデルですべてのロールを兼任させない**ことです。仮に 70B クラスのモデルを Autocomplete に充ててしまうと、<glossary:推論>が遅すぎて補完がまったく実用にならず、開発者は補完機能を切ってしまいます。逆に、補完用の 1.5B モデルで Chat を運用すると、説明の品質が物足りなくなります。

### 実運用向けの `config.yaml` 例

以下は、Chat / Edit / Apply / Autocomplete / Embed の各ロールに別々のモデルを割り当てた、典型的な実運用構成です。

```yaml
# ~/.continue/config.yaml  ——  実運用向けの構成例

models:
  # --- Chat / Edit 用の大型モデル ---
  - name: Chat（大型・高品質）
    provider: openai
    model: <your-large-chat-model>          # 例: qwen2.5-coder-32b
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - chat
      - edit

  # --- Apply 用の中型モデル ---
  - name: Apply（中型・高速）
    provider: openai
    model: <your-mid-model>                 # 例: qwen2.5-coder-7b
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - apply

  # --- Autocomplete 専用モデル（roles: [autocomplete] で指定）---
  - name: Autocomplete（小型・低レイテンシ）
    provider: ollama
    model: <your-autocomplete-model>          # 例: qwen2.5-coder:1.5b
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - autocomplete

  # --- Embedding 専用モデル（10.5 で詳細）---
  - name: Embeddings
    provider: ollama
    model: <your-embedding-model>           # 例: nomic-embed-text
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - embed
```

ポイントは次の 3 点です。

1. **`models` の各エントリに `roles` を明示する** — 1 つのモデルを複数ロールで兼任させる場合は、リストで列挙します（例: `chat` と `edit`）。Embedding 専用モデルのように 1 ロール限定の場合は単独で書きます。
2. **Apply ロールを意識的に分離する** — Chat や Edit の応答を実際のファイルに反映させる「Apply」ロールは、書き換え量が多くなる傾向があり、大型モデルだと適用に時間がかかります。中型モデルへ割り振ると体感が大きく改善します。
3. **Autocomplete も `models` リストで管理** — `roles: [autocomplete]` を指定したエントリを `models` リストに追加します。

### モデルサイズ・VRAM・同時実行数のバランス

社内 GPU サーバのリソースは有限です。「いくつのモデルを同時にホストできるか」は VRAM 容量と<glossary:量子化>レベルで決まります。代表的な目安を示します。

| モデル規模 | 量子化 | おおよその VRAM | 想定ロール |
| --- | --- | --- | --- |
| 1.5B〜3B | INT4 (Q4) | 2〜4 GB | Autocomplete |
| 7B | INT4 (Q4) | 5〜7 GB | Apply、軽量 Chat |
| 14B〜32B | INT4 (Q4) | 10〜20 GB | Chat、Edit |
| 70B | INT4 (Q4) | 40〜45 GB | 大規模 Chat、Edit |
| Embedding（〜500M） | FP16 | 1〜2 GB | Embed |

!!! tip "1 台のサーバに詰め込みすぎない"
    Chat 用の大型モデルと Autocomplete 用の小型モデルを同じ GPU 1 枚に同居させると、Chat の推論中に補完がブロックされる事象がしばしば起こります。サーバを物理的に分けるか、少なくとも別プロセス（別ポート）で起動し、Continue 側からは別<glossary:エンドポイント>として見えるようにすると、運用が安定します。

### 複数モデルの共存とフォールバック

複数の Chat モデルを `models` に登録しておくと、Continue のチャットパネル上部のドロップダウンから切り替えられます。社内では「日常用の中型モデル」と「難しい質問用の大型モデル」を併存させて、用途に応じて使い分けるパターンがよく見られます。

```yaml
models:
  - name: 日常用（軽量・高速）
    provider: openai
    model: <your-mid-chat-model>
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - chat
      - edit

  - name: 難しい質問用（大型・高品質）
    provider: openai
    model: <your-large-chat-model>
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - chat
```

!!! note "本構成では使用しません"
    クラウド LLM をフォールバック先として登録する構成もありますが、エアギャップ環境では使用しません。すべてのモデルは社内 LLM サーバ上で動作するものに限定してください。

---

## `@codebase` でコードベース全体をコンテキスト化する

`@codebase` は、現在 VS Code で開いている<glossary:ワークスペース>全体を検索し、質問に関連するコードを自動でコンテキストとして抽出する機能です。第 6 章で使い方の概要に触れましたが、本節では仕組みと運用面を掘り下げます。

### `@codebase` の仕組み

`@codebase` の内部処理は、大きく次のステップに分かれます。

1. **<glossary:チャンク分割>**: リポジトリ内のソースファイルを、関数や構文単位で適度な大きさ（典型的には数百<glossary:トークン>）に分割します
2. **Embedding 生成**: 各チャンクを Embedding モデル（10.5 で詳述）に通し、意味を表す高次元ベクトルへ変換します
3. **インデックス保存**: 生成されたベクトルをローカルのベクトルデータベースに保存します（ファイルパス、行範囲、ハッシュとセットで）
4. **検索**: 質問文も同じ Embedding モデルでベクトル化し、ベクトル類似度の高いチャンクを上位 N 件取得します
5. **（任意で）再ランキング**: Reranker モデルで関連度を再評価し、より精度の高い順序へ並べ替えます
6. **コンテキスト注入**: 取得したチャンクを `@codebase` の参照内容として LLM の<glossary:プロンプト>に添えます

このフローはすべてローカルで完結します。**エアギャップ環境でもコードが外部に送信されることはありません**（第 4 章で扱った通り、<glossary:テレメトリ>無効化と組み合わせて初めて保証されます）。

### インデックスの保存場所

Continue は `@codebase` のインデックスをユーザーごとのキャッシュディレクトリに保存します。

| OS | デフォルトの保存場所 |
| --- | --- |
| Linux / macOS | `~/.continue/index/` |
| Windows | `%USERPROFILE%\.continue\index\` |

中身はワークスペース単位のサブディレクトリと、SQLite ベースのインデックスファイル、ベクトルデータベースのファイルで構成されます。

```bash
# Linux / macOS でインデックスのサイズを確認する例
du -sh ~/.continue/index/

# Windows PowerShell の場合
Get-ChildItem $env:USERPROFILE\.continue\index -Recurse |
    Measure-Object -Property Length -Sum
```

ワークスペースを VS Code で初めて開いた直後、または `config.yaml` の Embedding モデル設定を変更した直後は、インデックスがバックグラウンドで構築されます。初回の構築には、リポジトリの規模に応じて数分〜数十分かかることがあります。

!!! tip "インデックス構築の進捗を確認する"
    VS Code のステータスバーにある Continue のアイコンにマウスオーバーすると、インデックス構築の進捗が表示されます。「Indexing: 45%」のような形で進行率が見え、完了するまで `@codebase` の精度が低い状態が続きます。

### 大規模リポジトリでの注意点

社内のモノレポは数十万ファイル規模になることがあります。すべてをインデックス対象にすると、構築時間・ディスク使用量・検索ノイズの 3 拍子で実用性が下がります。次の対処を組み合わせてください。

**1. `.continueignore` で対象を絞る**

ワークスペースのルートに `.continueignore` ファイルを置くと、`.gitignore` と同じ書式で対象外パターンを指定できます。

```text
# .continueignore  ——  インデックスから除外するパターン

# 自動生成ファイル
**/*.generated.*
**/__generated__/**
**/proto/**/*.pb.go

# 依存ライブラリのソース
node_modules/
vendor/
.venv/
build/
dist/

# 大きすぎるテストデータ
test/fixtures/large/
**/*.snapshot

# バイナリ・メディア
**/*.bin
**/*.pdf
**/*.png
**/*.jpg
```

`.gitignore` に書かれているパターンも Continue は基本的に尊重しますが、**`.continueignore` は「インデックスからのみ除外したい」追加パターン**を明示するのに使います。たとえば、Git では追跡したいが `@codebase` の検索対象には含めたくない自動生成ファイルなどに有効です。

**2. ワークスペースを分割する**

1 つの VS Code ウィンドウで巨大モノレポをすべて開くのではなく、関心領域ごとにワークスペースを分割するのも有効です。Continue のインデックスはワークスペース単位で作られるため、必要な範囲だけを対象にできます。

**3. ファイル拡張子で大胆に絞る**

`config.yaml` の補完設定で `disableInFiles` を使ったように、`@codebase` のインデックス対象も `.continueignore` で拡張子レベルから絞れます。社内のサーバサイドリポジトリで「Java と SQL だけインデックスしたい」のような場合は、それ以外を `.continueignore` で除外する方針が現実的です。

### インデックスを再構築するタイミング

次の変化があったときは、インデックスを破棄して再構築する方が安全です。

- Embedding モデルを別のものに切り替えた
- `.continueignore` を大幅に変更した
- リポジトリの構造を大規模に変更した（ディレクトリ全面再編、サブツリーの追加など）

再構築は、インデックスのキャッシュディレクトリを削除して VS Code を再起動するのが確実です。

```bash
# Linux / macOS の場合（必要なワークスペースのインデックスのみ削除する）
rm -rf ~/.continue/index/

# Windows PowerShell の場合
Remove-Item -Recurse -Force $env:USERPROFILE\.continue\index
```

!!! warning "全社共通の運用ルールを決めておく"
    インデックスはローカルキャッシュのため、各開発者のマシンで状態が異なります。Embedding モデル変更や `.continueignore` の改定は、社内の共通連絡経路（チャット・ポータル）で周知し、各自にインデックス再構築を依頼する運用にしておくと、品質のばらつきを抑えられます。

---

## `@docs` で社内ドキュメントをコンテキスト化する

`@codebase` がソースコードを対象にするのに対し、`@docs` は **ドキュメントサイトやマニュアル類** を対象にする機能です。社内 Wiki、社内フレームワークの API リファレンス、運用手順書などをインデックスしておくと、Chat で「この社内 API の使い方を教えて」と聞いたときに、実際の社内ドキュメントを根拠にした回答を得られます。

### `@docs` の仕組み

`@docs` の処理フローは `@codebase` に似ていますが、入力ソースが異なります。

1. **ソースの取得**: `config.yaml` の `docs` エントリで指定された URL またはパスから、HTML / Markdown / テキストファイルを取得します
2. **テキスト抽出**: HTML の場合はナビゲーション・ヘッダ・フッタを除去して本文だけを取り出します
3. **チャンク分割と Embedding**: `@codebase` と同じ要領でチャンクに分割し、Embedding モデルでベクトル化します
4. **インデックス保存**: 専用のインデックス領域に保存します（コードのインデックスとは分離されます）
5. **検索**: 質問に対して、登録済みの全 `docs` エントリを横断して類似度検索します
6. **コンテキスト注入**: 関連ドキュメントの本文と元 URL をプロンプトに添えます

### エアギャップ環境で `@docs` のソースをどう指定するか

`@docs` は本来クラウド型サービスのドキュメントサイト（公式の API リファレンスなど）を想定した機能ですが、エアギャップ環境では **社内 HTTP サーバ・ローカルパス・事前にエクスポートしたファイル** のいずれかから取り込みます。

| ソース形態 | 指定例 | 向いている用途 |
| --- | --- | --- |
| 社内 HTTP サーバ | `http://<your-internal-docs-host>/api-reference/` | 社内ポータルにホスティングされた API リファレンス・Wiki エクスポート |
| ローカルファイル（HTML） | `file:///srv/docs/internal-framework/index.html` | サイトとしてビルド済みのドキュメントを社内配布した場合 |
| ローカルファイル（Markdown 群） | `file:///srv/docs/handbook/` | リポジトリからエクスポートした Markdown 集 |

!!! warning "外部 URL は指定しない"
    `@docs` は URL を見ると自動的にクロールを試みます。インターネット上のドキュメントサイト（外部公式サイトなど）を `docs` エントリへ記述すると、エアギャップ前提が崩れます。**必ず社内ホストかローカルパスのみを指定**してください。指定先のホスト名・パスは、社内のレビューを経たものに限定する運用が望ましいです。

### `config.yaml` の `docs` 設定例

`docs` は `config.yaml` のトップレベルに、エントリ単位で配列として記述します。

```yaml
# ~/.continue/config.yaml  ——  @docs の設定例

docs:
  # 社内フレームワーク InternalFw の API リファレンス
  - name: InternalFw API Reference
    startUrl: "http://<your-internal-docs-host>/internalfw/api/"
    rootUrl: "http://<your-internal-docs-host>/internalfw/api/"

  # 社内コーディング規約（社内 HTTP サーバにビルド済みサイトを配置）
  - name: 社内コーディング規約
    startUrl: "http://<your-internal-docs-host>/coding-standards/"
    rootUrl: "http://<your-internal-docs-host>/coding-standards/"

  # ローカルにダンプ済みの運用手順書（Markdown 群）
  - name: 運用手順書
    startUrl: "file:///srv/docs/runbooks/index.md"
    rootUrl: "file:///srv/docs/runbooks/"
```

各キーの意味は次のとおりです。

| キー | 役割 |
| --- | --- |
| `name` | `@docs` で選択する際に表示される名前。日本語可。一意であること |
| `startUrl` | クロールの起点となる URL またはローカルパス |
| `rootUrl` | クロール範囲の上限。`startUrl` と同じドメイン・サブツリーに限定するために指定 |

`rootUrl` を指定することで、`@docs` が `startUrl` の階層から外に出てクロールするのを防げます。エアギャップ環境では特に、想定外の URL を辿らないように **`rootUrl` を明示する習慣**を付けておくと安全です。

### `@docs` を Chat から使う

設定後、Chat の入力欄で `@docs` と入力すると、登録済みのドキュメント名が一覧表示されます。対象を選択して質問を続けます。

```text
@docs InternalFw API Reference

InternalFw の RetryPolicy クラスでバックオフ間隔をジッタ付きで設定する方法を教えてください。
```

複数の `docs` エントリを同時に参照させたい場合は、`@docs` を続けて指定します。

```text
@docs InternalFw API Reference
@docs 社内コーディング規約

InternalFw の RetryPolicy を使う際に、社内規約に沿った命名と例外処理の書き方の例を示してください。
```

### インデックスの管理と更新運用

`@docs` のインデックスも `~/.continue/index/` 配下に保存されます。ドキュメントの内容が更新された場合は、Continue がいつ再クロールするかを把握しておく必要があります。

- **初回**: `config.yaml` に新しい `docs` エントリを追加して保存した直後に、バックグラウンドでクロールとインデックス構築が走ります
- **更新時の挙動**: デフォルトでは自動的な定期再クロールは行われません。ドキュメントが更新されたら、ユーザーが明示的に再クロールを促す必要があります
- **再クロール方法**: VS Code の<glossary:コマンドパレット>（`Ctrl+Shift+P` / `Cmd+Shift+P`）から `Continue: Reindex Docs` を実行するか、対象 `docs` エントリの設定を一度削除して保存→再追加することで再構築されます

!!! tip "ドキュメント更新の運用フロー"
    社内ドキュメントを集約管理しているチームがある場合、ドキュメント更新時に「再クロールを推奨」とアナウンスする運用にしておくと、開発者全員のローカルインデックスが古いままになるのを防げます。バージョン番号（例: v2024.05.21）を `name` に含めるルールにしておくと、古いインデックスが残っていることに気付きやすくなります。

---

## ローカル Embedding / Reranker モデルを設定する

`@codebase` と `@docs` の検索品質は、Embedding モデル（意味ベクトル化）と Reranker モデル（再ランキング）の選定に強く依存します。本節では、エアギャップ環境で両者をローカル運用する方法を扱います。

### Embedding モデルの役割

Embedding モデルは、テキスト（コードチャンクやドキュメント本文）を **数百〜数千次元のベクトル** に変換する専用モデルです。Chat モデルと違って文章を生成することはなく、入力されたテキストの「意味」を空間上の位置として返すだけの軽量なモデルです。

Continue はこのベクトルを使って、質問と類似度の高いチャンクを検索します。つまり、Embedding モデルの品質が `@codebase` と `@docs` の精度の土台になります。

### Embedding モデルの選び方

エアギャップ環境で使える、代表的なオープン Embedding モデルを示します。いずれもオフライン入手・ローカル実行が可能です。

| モデル | パラメータ規模 | 主な特徴 |
| --- | --- | --- |
| `nomic-embed-text` | 約 137M | 軽量・高速。汎用テキストに強い。<glossary:Ollama> で標準的に動作 |
| `bge-m3` | 約 568M | 多言語対応。日本語ドキュメントが多い社内環境で有利 |
| `e5-mistral-7b-instruct` | 約 7B | 大型・高精度。VRAM に余裕がある場合 |

選定の指針は次のとおりです。

- **社内ドキュメントが日本語中心**: `bge-m3` のような多言語対応モデルが無難
- **コードが中心で英語コメントが多い**: `nomic-embed-text` で十分な精度が出ることが多い
- **検索精度を最優先する**: 大型モデルを試す価値はあるが、インデックス構築時間と VRAM 消費とのトレードオフを評価する

!!! note "モデルの入手はオフラインで"
    上記モデルは公開モデルですが、エアギャップ環境では事前にミラー／オフライン配布した上で社内 Ollama や <glossary:vLLM> 上にロードしておく必要があります。インターネット経由でのモデルダウンロードは行いません。

### `config.yaml` への登録例

10.2 で示した実運用構成の `models` リストには、すでに `roles: [embed]` で Embedding モデルを 1 つ登録していました。改めて Embedding 登録部分だけを切り出すと、次のようになります。

```yaml
models:
  # ...（chat, edit, apply, autocomplete のモデルは省略）...

  - name: Embeddings (nomic-embed-text)
    provider: ollama
    model: nomic-embed-text
    apiBase: "http://<your-internal-llm-host>:11434"
    roles:
      - embed
```

vLLM 上で Embedding モデルをホストしている場合は、OpenAI 互換の Embedding エンドポイント（`/v1/embeddings`）を `provider: openai` で指定します。

```yaml
  - name: Embeddings (bge-m3 on vLLM)
    provider: openai
    model: <your-embedding-model>          # 例: bge-m3
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - embed
```

保存すると Continue は新しい Embedding モデルを認識し、インデックスを自動再構築します。**Embedding モデルを切り替えると、それ以前に作られたインデックスは原則無効**になります（次元数や意味空間が変わるため）。再構築の進捗はステータスバーのアイコンで確認できます。

### Reranker モデルとは

Reranker は、Embedding ベースの検索で得られた候補（例: 上位 30 件）を、より精度の高い別モデルで **関連度の高い順に並べ替える** ためのモデルです。Embedding は「ベクトル距離」しか見ませんが、Reranker は質問とチャンクのペアをまとめて読み込み、文脈に応じた関連度スコアを返します。

検索ステップは以下のように 2 段構えになります。

1. **粗い絞り込み**（Embedding）: 数十万チャンクから類似度上位 30 件を素早く取得
2. **精緻な並べ替え**（Reranker）: 30 件を Reranker に通し、上位 5〜10 件を最終的なコンテキストとして使用

### Reranker を導入するかの判断基準

Reranker は追加の VRAM とレイテンシを消費します。次の条件に当てはまる場合は導入を検討する価値があります。

- `@codebase` や `@docs` の回答で「関連が薄いコードや文書が混ざる」と感じる場面が多い
- インデックス対象のリポジトリが大規模で、Embedding 単独では候補のノイズが大きい
- Chat の応答速度より、回答の正確性を優先したい

逆に、次のような場合は Reranker なしで運用する方が現実的です。

- 補完中心の使い方で、Chat の `@codebase` 利用頻度が低い
- 社内 GPU リソースに余裕がない
- ドキュメント数・コード規模が小さく、Embedding 単独でも十分絞り込めている

### Reranker の `config.yaml` 例

エアギャップ環境で動かしやすい Reranker としては、`bge-reranker` シリーズなどが知られています。`roles: [rerank]` を指定して `models` リストに追加します。

```yaml
models:
  # ...（chat, edit, apply, autocomplete, embed のモデルは省略）...

  - name: Reranker (bge-reranker)
    provider: openai
    model: <your-reranker-model>           # 例: bge-reranker-v2-m3
    apiBase: "http://<your-internal-llm-host>:<port>/v1"
    apiKey: "dummy"
    roles:
      - rerank
```

!!! warning "Reranker は任意機能"
    Reranker を登録していなくても `@codebase` と `@docs` は機能します。まず Embedding のみで運用してみて、ノイズが許容範囲を超えたと感じた段階で Reranker を追加する、という二段階アプローチが現実的です。

---

## `@codebase` と `@docs` の使い分け

仕組みを押さえたところで、実際の質問でどちらを使えばよいかを整理します。両者は対立する機能ではなく、**「コードを根拠にしたいか、文書を根拠にしたいか」** の違いです。

### 判断フロー

質問を受けたときに、次の順で判断するとぶれにくくなります。

1. **質問の答えが、社内のソースコードに「実装として書かれている」か？**
    - はい → `@codebase`（または `@file` で対象ファイルを絞る）
    - いいえ → 次へ
2. **質問の答えが、社内のドキュメント（API リファレンス、運用手順、規約）に「説明として書かれている」か？**
    - はい → `@docs`
    - いいえ → 次へ
3. **コードと文書の両方を見ないと答えが出ない質問か？**
    - はい → `@codebase` と `@docs` を併用
    - いいえ → `@file` でピンポイントに対象を指定するか、コンテキストなしでまず質問してみる

### 典型的なユースケース

| 質問のタイプ | 推奨コンテキスト | 例 |
| --- | --- | --- |
| 既存実装の挙動を知りたい | `@codebase` | 「決済処理で例外をキャッチしているのはどのファイル？」 |
| 既存実装に変更を加えたい | `@file` または `@codebase` | 「`PaymentService.process()` をリトライ対応に書き換えたい」 |
| API の正しい呼び出し方を知りたい | `@docs` | 「InternalFw の RetryPolicy のオプションを教えて」 |
| 社内規約に沿った書き方を知りたい | `@docs` | 「社内コーディング規約での命名ルールは？」 |
| 規約に沿って実装を直したい | `@codebase` ＋ `@docs` | 「このサービスのリトライ実装を社内規約の RetryPolicy に置き換えて」 |
| 障害対応・運用手順を知りたい | `@docs` | 「DB レプリカ昇格の手順を教えて」 |

### 併用のコツ

`@codebase` と `@docs` を同時に使う場合は、**「両方の検索結果を LLM のコンテキストに詰め込む」**ことになります。コンテキスト長には上限があるため、いずれかが大量に取られると他方が削られます。以下を意識すると無駄が減ります。

- 質問は「コード側の対象」と「文書側の対象」を明示する。例: 「`PaymentService` のリトライ実装を、`@docs InternalFw API Reference` の RetryPolicy に置き換えて」
- 不要な `docs` エントリを `@docs` 指定から外す。Continue は指定された `docs` のみを検索対象にする
- 一度の質問で 3 つも 4 つも `@docs` を並べない。多すぎるとノイズが増える

### 使い分けのアンチパターン

次のような使い方は、よい結果を生みにくい傾向があります。

- **`@codebase` の万能視**: 「ドキュメントに書いてあること」をコードから推定させようとすると、<glossary:ハルシネーション>が増えます。文書ベースの質問は `@docs` を使う方が安全です（第 6 章でも触れたとおり、ハルシネーション抑止にはコンテキストの質が重要です）
- **常に全部入り**: 「どんな質問でも `@codebase` ＋ `@docs` を全エントリ指定」とすると、検索ノイズが増え、応答も遅くなります。質問ごとに必要なコンテキストだけを選ぶ習慣をつけます
- **巨大ファイルを `@file` で丸ごと**: 1 ファイル数千行を `@file` で渡すよりも、`@codebase` に任せて関連部分だけ抽出してもらう方が、コンテキスト効率が良いことがあります

---

## カスタムコンテキストプロバイダを作る

`@codebase` や `@docs` で取り込めない情報源を Chat に渡したい場面があります。たとえば、社内 Issue トラッカーからエクスポートした JSON、社内 CI のビルドログ、社内データベースのスキーマ定義などです。Continue は **カスタムコンテキストプロバイダ** という仕組みを提供しており、`@<任意の名前>` で参照できる情報源を自作できます。

### コンテキストプロバイダの位置づけ

`@file` `@codebase` `@docs` `@terminal` などは、すべて「コンテキストプロバイダ」の実装の 1 つです。`config.yaml` の `context` リストへの記述で有効化／無効化されます。カスタムコンテキストプロバイダは、ここに **自作のプロバイダ** を追加するイメージです。

実装方法はいくつかありますが、本節ではエアギャップ環境で扱いやすい **HTTP コンテキストプロバイダ** を例にします。これは Continue が指定の URL に HTTP リクエストを送り、返ってきた JSON をコンテキストとしてプロンプトへ添える仕組みです。社内 LAN 内に小さな HTTP サーバを立てれば、社内独自の情報源を `@<好きな名前>` から呼び出せます。

!!! note "外部 API は使用しません"
    HTTP コンテキストプロバイダの参照先 URL は、**必ず社内 LAN 内のホストに限定** してください。インターネット上のサービスを参照すると、エアギャップ前提が崩れます。

### ハンズオン: 社内 Issue を取り込むコンテキストプロバイダを作る

ここでは、社内 Issue 一覧の JSON エクスポートを返す、最小限の社内 HTTP サーバを Python で立て、Continue から `@issues` で呼び出すまでを体験します。

#### ステップ 1: 社内 Issue の JSON ダンプを用意する

開発マシン上で、社内 Issue から抜粋した JSON ファイルを用意します（実運用では社内 Issue トラッカーから定期エクスポートしてください）。

```bash
# 作業用ディレクトリ
mkdir -p /tmp/continue-context-demo && cd /tmp/continue-context-demo

cat > issues.json <<'EOF'
[
  {
    "id": "INT-1042",
    "title": "決済リトライ時にロックが解放されないことがある",
    "status": "open",
    "owner": "<your-team-member>",
    "summary": "PaymentService.process() で例外発生時にロックの finally 解放漏れの可能性あり"
  },
  {
    "id": "INT-1051",
    "title": "InternalFw のリトライ実装を全社で統一",
    "status": "in-progress",
    "owner": "<your-team-member>",
    "summary": "ad-hoc な while-retry を InternalFw RetryPolicy へ統一する全社タスク"
  }
]
EOF
```

#### ステップ 2: コンテキスト用 HTTP サーバを Python で立てる

Continue の HTTP コンテキストプロバイダは、`POST` で `query`（ユーザーがコンテキストに添える検索キーワード）を受け取り、`name` と `content` を含む配列を JSON で返すサーバを期待します。最小実装は次のとおりです。

```python
# /tmp/continue-context-demo/server.py
import json
from http.server import BaseHTTPRequestHandler, HTTPServer

ISSUES_PATH = "/tmp/continue-context-demo/issues.json"


class IssueHandler(BaseHTTPRequestHandler):
    def do_POST(self):  # noqa: N802
        length = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(length).decode("utf-8") if length else "{}"
        payload = json.loads(body or "{}")
        query = (payload.get("query") or "").lower()

        with open(ISSUES_PATH, encoding="utf-8") as f:
            issues = json.load(f)

        # クエリ文字列で簡易フィルタ（指定がなければ全件返す）
        if query:
            issues = [
                i for i in issues
                if query in i["title"].lower() or query in i["summary"].lower()
            ]

        # Continue が期待する形式: [{ "name": ..., "description": ..., "content": ... }, ...]
        result = [
            {
                "name": f"{issue['id']} {issue['title']}",
                "description": f"status={issue['status']} owner={issue['owner']}",
                "content": (
                    f"# {issue['id']} — {issue['title']}\n"
                    f"- status: {issue['status']}\n"
                    f"- owner: {issue['owner']}\n\n"
                    f"{issue['summary']}\n"
                ),
            }
            for issue in issues
        ]

        body_out = json.dumps(result, ensure_ascii=False).encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body_out)))
        self.end_headers()
        self.wfile.write(body_out)


if __name__ == "__main__":
    # 開発マシン上で待ち受け
    HTTPServer(("127.0.0.1", 8765), IssueHandler).serve_forever()
```

別ターミナルでサーバを起動します。

```bash
python /tmp/continue-context-demo/server.py
# => http://127.0.0.1:8765 で待ち受け開始
```

簡単な疎通確認をしておきます。

```bash
curl -s -X POST http://127.0.0.1:8765/ \
  -H "Content-Type: application/json" \
  -d '{"query": "リトライ"}' | head -c 200
```

#### ステップ 3: `config.yaml` にカスタムコンテキストプロバイダを登録する

`contextProviders` に、HTTP プロバイダのエントリを追加します。

```yaml
context:
  # 既存のプロバイダ（第 5 章で導入したもの）
  - provider: code
  - provider: docs
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: codebase

  # 追加: 社内 Issue を取り込むカスタムプロバイダ
  - provider: http
    params:
      url: "http://127.0.0.1:8765/"
      title: "issues"           # @issues で呼び出せるようにする
      displayTitle: "社内 Issue"
      description: "社内 Issue トラッカーから関連 Issue を取り込む"
```

`config.yaml` を保存すると、Continue が新しいプロバイダを読み込みます。

#### ステップ 4: Chat から呼び出す

Chat パネルを開き、入力欄で `@issues` と入力すると、登録したカスタムプロバイダが候補に現れます。クエリを添えて質問してみましょう。

```text
@issues リトライ

決済処理のリトライ実装を改善したいです。関連する社内 Issue を踏まえて、まず確認すべきポイントを 3 つ挙げてください。
```

サーバ側にリクエストが届き、`issues.json` の中から「リトライ」を含む Issue だけがコンテキストとして返ります。LLM はそれを踏まえて、社内文脈に即した助言を生成します。

#### ステップ 5: 終了処理

検証が終わったら、Python サーバ（`Ctrl+C`）を停止し、必要に応じてサンプルファイルを削除してください。

```bash
rm -rf /tmp/continue-context-demo
```

### カスタムプロバイダ運用時の留意点

エアギャップ環境で自作プロバイダを本番運用するときは、次の観点に注意してください。

- **アクセス制御**: コンテキスト用 HTTP サーバは社内 LAN 限定でリッスンし、認証が必要な場合は社内標準の認証方式（社内 SSO、相互 TLS など）と組み合わせる
- **機密情報の選別**: Issue や設計文書には、開発者本人がアクセスしてよい範囲を超える情報が含まれることがあります。プロバイダ側のフィルタで、利用者の権限に応じた絞り込みを行う
- **PII の混入回避**: 個人情報・顧客情報を含むデータをそのままコンテキストに返さない。マスキング処理をプロバイダ側で施す
- **ログの設計**: どの利用者がいつ何を取得したかをサーバ側で記録し、<glossary:監査ログ>として残せるようにする（第 13 章で扱う運用観点と連動）
- **データの更新**: 社内 Issue トラッカーからのエクスポートを定期実行するスケジュールを決め、古いダンプを参照し続ける状態を避ける

!!! tip "プロバイダの粒度を分ける"
    `@issues` だけでなく、`@runbooks`（運用手順）、`@adr`（設計判断記録）、`@schema`（DB スキーマ）のように、用途ごとに別プロバイダとして登録する方が、利用者にとってわかりやすくなります。それぞれを別 HTTP エンドポイントとして実装し、`contextProviders` に並べて登録する形が運用しやすいです。

---

## まとめ

- LLM の出力品質は「どのモデルを使うか」と「どんなコンテキストを渡すか」の両輪で決まり、エアギャップ環境では後者の設計が特に効きます
- `models` の各エントリに `roles` を明示し、Chat / Edit / Apply / Autocomplete / Embed / Rerank を用途ごとのモデルに振り分けると、レイテンシと品質のバランスが取れます。Autocomplete も `roles: [autocomplete]` で `models` リストに登録します
- `@codebase` はローカルで完結するベクトル検索で、`.continueignore` とインデックス再構築の運用ルールを併せて決めることが実用化の鍵です
- `@docs` のソースは必ず社内 HTTP サーバまたはローカルパスに限定し、`rootUrl` でクロール範囲を明示します。インデックス更新は明示的に促す運用が現実的です
- ローカル Embedding モデル（例: `nomic-embed-text`、`bge-m3`）を `roles: [embed]` で登録すると `@codebase` と `@docs` が機能します。Reranker は任意機能で、ノイズが気になった段階で追加します
- 社内独自の情報源は、HTTP コンテキストプロバイダとして自作することで `@<好きな名前>` で参照できます。アクセス制御・機密情報の選別・監査ログを必ず併せて設計します

---

## 次の章へ

次は [第 11 章 カスタマイズで開発フローに最適化する](11-customization.md) で、社内コーディング規約を Rules として落とし込んだり、カスタムスラッシュコマンドで定型作業をワンクリック化したりする方法を扱います。本章で「社内の文脈」をモデルに渡す土台を整えた上で、次章では「社内の作法」をモデルに守らせるカスタマイズへと進みます。

---

## 参考リンク

- [Continue 公式ドキュメント — Codebase Context](https://docs.continue.dev/customize/deep-dives/codebase)
- [Continue 公式ドキュメント — @docs Context Provider](https://docs.continue.dev/customize/context-providers#docs)
- [Continue 公式ドキュメント — Context Providers](https://docs.continue.dev/customize/context-providers)
- [Continue 公式ドキュメント — Embeddings Models](https://docs.continue.dev/customize/model-roles/embeddings)
- [Continue 公式ドキュメント — Reranking Models](https://docs.continue.dev/customize/model-roles/reranker)
- [Nomic Embed Text モデルカード（Hugging Face）](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5)
- [BGE-M3 モデルカード（Hugging Face）](https://huggingface.co/BAAI/bge-m3)
- [BGE Reranker モデルカード（Hugging Face）](https://huggingface.co/BAAI/bge-reranker-v2-m3)
