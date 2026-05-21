# CLAUDE.md — Continue チュートリアル 編集ガイド

Claude Code がこのリポジトリを編集する際の規約・手順書です。

---

## プロジェクト概要

MkDocs + Material for MkDocs で構築した日本語技術ドキュメントです。  
VS Code 拡張 **Continue.dev** を **オンプレミス・エアギャップ環境** で利用するための、ソフトウェア開発者向けチュートリアルです。

### ファイル構造

```
docs/
  index.md                      # トップページ
  01-introduction.md            # 第1〜14章
  ...
  14-troubleshooting.md
  appendix/
    glossary.md                 # 用語集（ezglossary 形式）★
    config-reference.md         # 付録A
    shortcuts.md                # 付録B
mkdocs.yml                      # ビルド設定（ezglossary プラグイン含む）
CLAUDE.md                       # 本ファイル
```

---

## 用語集の管理（最重要）

用語集は **`docs/appendix/glossary.md`** で一元管理しています。  
[mkdocs-ezglossary-plugin](https://github.com/realtimeprojects/mkdocs-ezglossary) を使用しており、定義は 1 か所だけに書きます。

### 用語の定義フォーマット

```markdown
glossary:用語名
:   定義文。日本語で簡潔に記述する。
    英語名がある場合は定義冒頭に（英語: EnglishTerm）の形で含める。
    初出 → [第N章](../NN-slug.md)
```

**実例：**

```markdown
glossary:エアギャップ
:   ネットワーク上で完全に隔離された環境のことです（英語: Air-gap）。
    インターネットや外部ネットワークと物理的・論理的に遮断されており、
    外部との通信が一切できません。初出 → [第 1 章](../01-introduction.md)

glossary:LLM
:   大量のテキストデータを学習し、自然言語での会話・コード生成・要約などを
    行う AI モデルの総称です（Large Language Model、大規模言語モデル）。
    初出 → [第 1 章](../01-introduction.md)
```

### 用語名の命名規則

| ケース | 命名例 |
|---|---|
| 英語の技術用語 | `LLM`, `MCP`, `VSIX`, `VRAM` |
| 日本語の概念 | `エアギャップ`, `量子化`, `推論` |
| スペースを含む複合語 | `MCPサーバ`, `Embeddingモデル`, `OpenAI互換API` |
| セクションプレフィックス | 常に `glossary:` を使用 |

**注意:** スペースなしで連結する（`MCP サーバ` → `MCPサーバ`）。

### 新しい用語を追加する手順

1. **`docs/appendix/glossary.md` を開く**

2. **適切なカテゴリ（A〜H）のセクションに追加する**

   ```
   A. Continue 機能・画面
   B. VS Code・拡張機能
   C. LLM・AI 全般
   D. config.yaml・設定
   E. コンテキスト・インデックス
   F. MCP（Model Context Protocol）
   G. エアギャップ・オンプレミス・セキュリティ
   H. ネットワーク・ローカル LLM サーバ
   ```

3. **以下の形式で追記する**（既存エントリの後に `---` で区切って追加）

   ```markdown
   ---

   glossary:新用語名
   :   定義文。（英語: EnglishTerm）を冒頭に含める。
       初出 → [第N章](../NN-slug.md)
   ```

4. **章ファイルの初出箇所にリンクを追加する**（次のセクション参照）

---

## 章ファイルからのリンク追加

ezglossary では `<glossary:用語名>` 構文で本文からリンクを張れます。

### 基本ルール

- **1ファイルにつき各用語 1 回のみ**（初出箇所のみ）
- コードブロック（` ``` `）内には書かない
- インラインコード（`` ` `` `` ` ``）内には書かない
- 見出し行（`# ...`）には書かない

### リンクの書き方

```markdown
# 変更前
エアギャップ環境では LLM をローカルサーバでホストします。

# 変更後（初出箇所のみ）
<glossary:エアギャップ>環境では <glossary:LLM> をローカルサーバでホストします。
```

### 用語名とリンク構文の対応表（主要用語）

| 本文中の表記 | リンク構文 |
|---|---|
| エアギャップ | `<glossary:エアギャップ>` |
| LLM | `<glossary:LLM>` |
| MCP | `<glossary:MCP>` |
| MCP サーバ | `<glossary:MCPサーバ>` |
| Autocomplete | `<glossary:Autocomplete>` |
| config.yaml | `<glossary:config.yaml>` |
| Embedding モデル | `<glossary:Embeddingモデル>` |
| OpenAI 互換 API | `<glossary:OpenAI互換API>` |
| ローカル操作型 MCP サーバ | `<glossary:ローカル操作型MCPサーバ>` |
| LM Studio | `<glossary:LMStudio>` |

---

## Markdown 記述規約

### フロントマター（全ファイル必須）

```yaml
---
title: "第N章 タイトル"
type: chapter        # chapter / appendix / index
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
---
```

### エアギャップ前提の注意事項

- 外部 URL を `@docs` の参照先に使用しない
- Google Fonts・Google Analytics などの外部リソースを参照しない
- コードサンプルで `pip install` する場合は社内ミラーからの取得を前提にする旨を注記する

---

## ビルド・動作確認

```bash
# 依存パッケージのインストール（社内ミラーが前提）
pip install mkdocs-material mkdocs-ezglossary-plugin

# ローカルプレビュー
mkdocs serve

# 静的サイト生成
mkdocs build
```

---

## よくある作業パターン

### 新しい章を追加する

1. `docs/NN-slug.md` を新規作成（フロントマター必須）
2. `mkdocs.yml` の `nav:` セクションに追記
3. 用語の初出箇所に `<glossary:用語名>` を追加

### 既存の用語定義を修正する

`docs/appendix/glossary.md` の該当エントリだけを編集する。  
（章ファイルのリンク構文 `<glossary:用語名>` はそのまま有効）

### 用語名を変更する場合

1. `glossary.md` の `glossary:旧名` を `glossary:新名` に変更
2. 全ファイルの `<glossary:旧名>` を `<glossary:新名>` に一括置換

   ```bash
   # 例: MCPServer → MCPサーバ に変更した場合
   find docs/ -name "*.md" -exec sed -i 's/<glossary:MCPServer>/<glossary:MCPサーバ>/g' {} +
   ```
