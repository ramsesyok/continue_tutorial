---
title: "第 10 章 社内モデルとコンテキスト管理"
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

- 社内向けに複数の LLM をロール別へ割り当てる、実運用視点での設計指針
- `@codebase` と `@docs` の仕組みと、エアギャップ環境でのインデックス運用
- ローカル Embedding モデル・Reranker モデルを `config.yaml` に組み込む方法
- 用途に応じた `@codebase` と `@docs` の使い分け
- カスタムコンテキストプロバイダを作って社内独自の情報源を取り込む手順（ハンズオン）

---

## 「社内モデル」と「コンテキスト管理」の関係

ここまでの章では、Chat（[第 6 章](06-chat-basics.md)）、Autocomplete（[第 7 章](07-autocomplete.md)）、Edit（[第 8 章](08-edit.md)）、Agent（[第 9 章](09-agent.md)）という、Continue の機能を 1 つずつ扱ってきました。これらに共通するのは、LLM への入力が **「どのモデルを使うか」** と **「どんなコンテキスト（情報）をモデルに渡すか」** の 2 つの軸で決まる、という事実です。

| 軸 | 何を決めるか | 主に扱う章 |
| --- | --- | --- |
| モデルの選定 | 速度・精度・VRAM のバランス、用途別の使い分け | 第 5 章・本章 10.2 |
| コンテキストの設計 | コードベース・社内文書・現在の編集状態を、どこまでモデルに見せるか | 第 6 章・本章 10.3〜10.7 |

この 2 つは独立しているように見えて、実運用では切っても切り離せません。高性能なモデルを用意しても、コンテキストとして渡される情報が貧しければ「自社の事情を知らない汎用回答」しか返ってきません。逆に豊富なコンテキストを渡せても、ロール別にモデルが適切に振り分けられていないと、補完が遅すぎたり、Chat の応答品質が足りなかったりします。

### エアギャップ環境ではコンテキスト管理が特に効く

クラウド型の AI コーディング支援サービスでは、公開リポジトリや一般公開ドキュメントの巨大な学習データに頼って「それなりの回答」を返せます。しかしオンプレミス・エアギャップ環境では、モデルが学習しているのは公開データだけです。社内独自のフレームワーク、社内 API、社内ライブラリの命名規則、社内のコーディング規約は、**コンテキストとして明示的に渡さない限り、モデルには見えません**。

つまり、エアギャップ環境で Continue を活用する鍵は「社内特有の情報をいかに効率よくモデルに渡すか」にあります。本章ではそのための仕組み（`@codebase`、`@docs`、ローカル Embedding、カスタムコンテキストプロバイダ）を順に扱います。

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

ここで重要なのは、**1 つの巨大モデルですべてのロールを兼任させない**ことです。仮に 70B クラスのモデルを Autocomplete に充ててしまうと、推論が遅すぎて補完がまったく実用にならず、開発者は補完機能を切ってしまいます。逆に、補完用の 1.5B モデルで Chat を運用すると、説明の品質が物足りなくなります。

### 実運用向けの `config.yaml` 例

以下は、Chat / Edit / Apply / Autocomplete / Embed の各ロールに別々のモデルを割り当てた、典型的な実運用構成です。

```yaml
# ~/.continue/config.yaml  ——  実運用向けの構成例

models:
  # --- Chat / Edit 用の大型モデル ---
  - name: Chat（大型・高品質）
    provider: openai
    model: <your-large-chat-model>          # 例: qwen2.5-coder-32b
    api_base: "http://<your-internal-llm-host>:<port>/v1"
    api_key: "dummy"
    roles:
      - chat
      - edit

  # --- Apply 用の中型モデル ---
  - name: Apply（中型・高速）
    provider: openai
    model: <your-mid-model>                 # 例: qwen2.5-coder-7b
    api_base: "http://<your-internal-llm-host>:<port>/v1"
    api_key: "dummy"
    roles:
      - apply

  # --- Embedding 専用モデル（10.5 で詳細）---
  - name: Embeddings
    provider: ollama
    model: <your-embedding-model>           # 例: nomic-embed-text
    api_base: "http://<your-internal-llm-host>:11434"
    roles:
      - embed

tabAutocompleteModel:
  name: Autocomplete（小型・低レイテンシ）
  provider: ollama
  model: <your-autocomplete-model>          # 例: qwen2.5-coder:1.5b
  api_base: "http://<your-internal-llm-host>:11434"
```

ポイントは次の 3 点です。

1. **`models` の各エントリに `roles` を明示する** — 1 つのモデルを複数ロールで兼任させる場合は、リストで列挙します（例: `chat` と `edit`）。Embedding 専用モデルのように 1 ロール限定の場合は単独で書きます。
2. **Apply ロールを意識的に分離する** — Chat や Edit の応答を実際のファイルに反映させる「Apply」ロールは、書き換え量が多くなる傾向があり、大型モデルだと適用に時間がかかります。中型モデルへ割り振ると体感が大きく改善します。
3. **Autocomplete は引き続き `tabAutocompleteModel`** — Autocomplete だけは別キーで指定するのが Continue の慣例です。`models` リストには入れません。

### モデルサイズ・VRAM・同時実行数のバランス

社内 GPU サーバのリソースは有限です。「いくつのモデルを同時にホストできるか」は VRAM 容量と量子化レベルで決まります。代表的な目安を示します。

| モデル規模 | 量子化 | おおよその VRAM | 想定ロール |
| --- | --- | --- | --- |
| 1.5B〜3B | INT4 (Q4) | 2〜4 GB | Autocomplete |
| 7B | INT4 (Q4) | 5〜7 GB | Apply、軽量 Chat |
| 14B〜32B | INT4 (Q4) | 10〜20 GB | Chat、Edit |
| 70B | INT4 (Q4) | 40〜45 GB | 大規模 Chat、Edit |
| Embedding（〜500M） | FP16 | 1〜2 GB | Embed |

!!! tip "1 台のサーバに詰め込みすぎない"
    Chat 用の大型モデルと Autocomplete 用の小型モデルを同じ GPU 1 枚に同居させると、Chat の推論中に補完がブロックされる事象がしばしば起こります。サーバを物理的に分けるか、少なくとも別プロセス（別ポート）で起動し、Continue 側からは別エンドポイントとして見えるようにすると、運用が安定します。

### 複数モデルの共存とフォールバック

複数の Chat モデルを `models` に登録しておくと、Continue のチャットパネル上部のドロップダウンから切り替えられます。社内では「日常用の中型モデル」と「難しい質問用の大型モデル」を併存させて、用途に応じて使い分けるパターンがよく見られます。

```yaml
models:
  - name: 日常用（軽量・高速）
    provider: openai
    model: <your-mid-chat-model>
    api_base: "http://<your-internal-llm-host>:<port>/v1"
    api_key: "dummy"
    roles:
      - chat
      - edit

  - name: 難しい質問用（大型・高品質）
    provider: openai
    model: <your-large-chat-model>
    api_base: "http://<your-internal-llm-host>:<port>/v1"
    api_key: "dummy"
    roles:
      - chat
```

!!! note "本構成では使用しません"
    クラウド LLM をフォールバック先として登録する構成もありますが、エアギャップ環境では使用しません。すべてのモデルは社内 LLM サーバ上で動作するものに限定してください。

---

## `@codebase` でコードベース全体をコンテキスト化する

`@codebase` は、現在 VS Code で開いているワークスペース全体を検索し、質問に関連するコードを自動でコンテキストとして抽出する機能です。第 6 章で使い方の概要に触れましたが、本節では仕組みと運用面を掘り下げます。

### `@codebase` の仕組み

`@codebase` の内部処理は、大きく次のステップに分かれます。

1. **チャンク分割**: リポジトリ内のソースファイルを、関数や構文単位で適度な大きさ（典型的には数百トークン）に分割します
2. **Embedding 生成**: 各チャンクを Embedding モデル（10.5 で詳述）に通し、意味を表す高次元ベクトルへ変換します
3. **インデックス保存**: 生成されたベクトルをローカルのベクトルデータベースに保存します（ファイルパス、行範囲、ハッシュとセットで）
4. **検索**: 質問文も同じ Embedding モデルでベクトル化し、ベクトル類似度の高いチャンクを上位 N 件取得します
5. **（任意で）再ランキング**: Reranker モデルで関連度を再評価し、より精度の高い順序へ並べ替えます
6. **コンテキスト注入**: 取得したチャンクを `@codebase` の参照内容として LLM のプロンプトに添えます

このフローはすべてローカルで完結します。**エアギャップ環境でもコードが外部に送信されることはありません**（第 4 章で扱った通り、テレメトリ無効化と組み合わせて初めて保証されます）。

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
