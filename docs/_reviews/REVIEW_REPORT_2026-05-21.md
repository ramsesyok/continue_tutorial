---
title: "横断レビューレポート 2026-05-21"
type: review
tags:
  - review
  - continue
  - vscode
  - on-premise
  - air-gap
date: 2026-05-21
reviewer: agent
---

# 横断レビューレポート 2026-05-21

## 1. レビュー概要

| 項目 | 内容 |
| --- | --- |
| 実施日 | 2026-05-21 |
| レビュー範囲 | `docs/` 配下の全 Markdown ファイル（第 1〜14 章・付録 A〜C・index.md） |
| 対象ファイル数 | 18 ファイル |
| 依拠した手順書 | `REVIEW_INSTRUCTIONS.md`・`AGENT_INSTRUCTIONS.md` |
| High（リリース前必須修正） | **2 件** |
| Medium（次回更新で修正） | **7 件** |
| Low（余裕があれば修正） | **1 件** |

---

## 2. サマリー

| 重大度 | 件数 | 主な内容 |
| --- | --- | --- |
| High | 2 | config キー名の不統一（`api_base` vs `apiBase`）、VS Code バージョン要件の不一致 |
| Medium | 7 | フロントマタータイトルの表記揺れ（5 ファイル）、Rules パスの誤記、埋め込みモデル設定スキーマの不一致、mcpServers スキーマの不一致、画像ファイルの欠落 |
| Low | 1 | 「次の章へ」リンクテキストの表記揺れ（第 4 章） |

---

## 3. 指摘事項詳細

### H-1 【High】`api_base` と `apiBase` の混在

- **カテゴリ**: 3.7 コードブロック・設定例の整合性
- **影響ファイル**:

| ファイル | 使用キー名 |
| --- | --- |
| `docs/05-config-yaml-basics.md` | `api_base`（スネークケース） |
| `docs/10-internal-models-context.md` | `api_base`（スネークケース） |
| `docs/appendix/config-reference.md` | `api_base`（スネークケース） |
| `docs/02-prerequisites.md` | `apiBase`（キャメルケース） |
| `docs/04-telemetry-airgap-verification.md` | `apiBase`（キャメルケース） |
| `docs/07-autocomplete.md` | `apiBase`（キャメルケース） |
| `docs/08-edit.md` | `apiBase`（キャメルケース） |
| `docs/09-agent.md` | `apiBase`（キャメルケース） |
| `docs/13-security-operations.md` | `apiBase`（キャメルケース） |
| `docs/14-troubleshooting.md` | `apiBase`（キャメルケース） |

- **問題**: `config.yaml` の設定キーはどちらか一方のみが正しい。読者がコードブロックをコピー＆ペーストした場合、誤った形式の記述は Continue に認識されず、接続エラーを引き起こす。
- **推奨対応**: Continue の公式ドキュメント（`https://docs.continue.dev`）でキー名を確認し、正しい表記に全ファイルを統一する。Continue v0.8 以降は YAML 形式で `apiBase`（キャメルケース）が標準とされているため、`api_base` 側を `apiBase` に修正することを推奨する。修正の際は付録 A（config リファレンス）も必ず合わせて更新すること。

---

### H-2 【High】VS Code バージョン要件の不一致

- **カテゴリ**: 3.8 章間整合性
- **影響ファイル**:
  - `docs/01-introduction.md`（行 98 付近）: "VS Code がインストールされている（バージョン **1.80 以降**を推奨）"
  - `docs/02-prerequisites.md`（行 29 付近）: "VS Code 1.85 以上"
- **問題**: 同じ「インストール済み VS Code のバージョン要件」に対して 2 つの異なる値が記載されている。読者が第 1 章を読んで 1.80 系を使い続けた場合、第 2 章で矛盾に気づいて混乱する。
- **推奨対応**: 実際に動作確認している最低バージョンを確定し、両ファイルを同一の値（例: "1.85 以上"）に統一する。

---

### M-1 【Medium】フロントマタータイトルの表記揺れ（数字・記号の前後スペース）

- **カテゴリ**: 3.2 フロントマター整合性
- **影響ファイル**（`AGENT_INSTRUCTIONS.md` の規則: 数字の前後に半角スペースを入れる）:

| ファイル | 現在の title | 正しい title |
| --- | --- | --- |
| `docs/01-introduction.md` | `"第1章 はじめに"` | `"第 1 章 はじめに"` |
| `docs/04-telemetry-airgap-verification.md` | `"第4章 テレメトリと外部通信の遮断確認"` | `"第 4 章 テレメトリと外部通信の遮断確認"` |
| `docs/05-config-yaml-basics.md` | `"第5章 config.yaml の基本構造"` | `"第 5 章 config.yaml の基本構造"` |
| `docs/06-chat-basics.md` | `"第6章 Chat 機能の基本"` | `"第 6 章 Chat 機能の基本"` |
| `docs/appendix/config-reference.md` | `"付録A config.yaml リファレンス"` | `"付録 A config.yaml リファレンス"` |

- **推奨対応**: 各ファイルのフロントマター `title:` 行を上表の「正しい title」に修正する。

---

### M-2 【Medium】Chapter 13 の Rules ファイルパスの誤記

- **カテゴリ**: 3.7 コードブロック・設定例の整合性
- **影響ファイル**: `docs/13-security-operations.md`（行 54 付近）
- **現在の記述**: `Rules（~/.continue/rules/ 配下のファイル）`
- **問題**: Chapter 11 および付録 A では、Rules は `config.yaml` の `rules:` フィールドまたはプロジェクトルートの `.continuerules` ファイルとして説明されている。`~/.continue/rules/` というディレクトリは Continue の標準的なファイル配置に存在しない。
- **推奨対応**: 記述を `Rules（config.yaml の rules フィールド、またはプロジェクトルートの .continuerules ファイル）` に修正し、第 11 章・付録 A との整合を取る。

---

### M-3 【Medium】埋め込みモデル設定スキーマの不一致

- **カテゴリ**: 3.7 コードブロック・設定例の整合性・3.8 章間整合性
- **影響ファイル**:
  - `docs/10-internal-models-context.md`: `models:` リスト内に `roles: [embed]` で埋め込みモデルを指定
  - `docs/14-troubleshooting.md`（行 236〜243）: `embeddingsProvider:` トップレベルキーを使用
  - `docs/appendix/config-reference.md`: `embeddingsProvider:` トップレベルキーを独立セクションとして記載
- **問題**: 2 つの異なる設定方法が混在しており、どちらが現在の Continue バージョンで有効かが不明確。読者が混乱する。
- **推奨対応**: 公式ドキュメントで現在推奨されている設定方法を確認し、全ファイルを統一する。旧形式と新形式の両方が存在する場合は、いずれかの章（例: 付録 A）でバージョン変更の注記を加える。

---

### M-4 【Medium】`mcpServers` スキーマの不一致

- **カテゴリ**: 3.7 コードブロック・設定例の整合性・3.8 章間整合性
- **影響ファイル**:
  - `docs/12-mcp-onprem.md`: フラット形式 `mcpServers: [{name: "x", command: "npx", args: [...]}]`
  - `docs/appendix/config-reference.md`: ネスト形式 `mcpServers: [{name: "x", transport: {type: "stdio", command: "node", args: [...]}}]`
- **問題**: 同じ `mcpServers` の設定例が 2 章で異なる構造を持っており、どちらが正しいか読者が判断できない。
- **推奨対応**: Continue の公式 MCP 設定ドキュメントを確認し、正しいスキーマに統一する。

---

### M-5 【Medium】画像ファイルの欠落

- **カテゴリ**: 3.10 画像ファイルの存在確認
- **影響ファイル**: `docs/03-offline-install.md`（行 96 付近）
- **参照パス**: `images/03/03-install-from-vsix-menu.png`
- **問題**: ファイルが `docs/images/03/` ディレクトリに存在しない（当該ディレクトリ自体が未作成）。ドキュメントビルド時に画像が表示されない。
- **推奨対応**: スクリーンショットを撮影して `docs/images/03/03-install-from-vsix-menu.png` に配置するか、該当の `![...]()` 記述をいったん削除してテキストによる説明に切り替える。

---

### L-1 【Low】「次の章へ」リンクテキストの表記揺れ

- **カテゴリ**: 3.5 章末構造・リンク整合性
- **影響ファイル**: `docs/04-telemetry-airgap-verification.md`
- **現在の記述**: `[第5章 config.yaml の基本構造](05-config-yaml-basics.md)` （リンクテキスト内で "第5章" とスペースなし）
- **問題**: リンク先ファイルや他章のリンクテキストは "第 5 章"（スペースあり）で統一されているが、ここだけ異なる。
- **推奨対応**: リンクテキストを `第 5 章 config.yaml の基本構造` に修正する。

---

## 4. 問題なし（良好）の確認事項

以下の項目は全ファイルを確認した結果、問題なしと判断しました。

| 確認項目 | 結果 |
| --- | --- |
| 「この章で学ぶこと」セクションの存在（全章） | ✅ 全章に存在 |
| 「まとめ」セクションの存在（全章） | ✅ 全章に存在 |
| 「次の章へ」セクションの存在（全章・最終章除く） | ✅ 全章に存在 |
| ★重点章のハンズオンセクション（第 4・7・8・9・10・11 章） | ✅ 全章に存在 |
| 第 4 章の監査証跡セクション（エアギャップ検証手順） | ✅ 存在 |
| `ローカルLLM`（スペースなし）の混入 | ✅ 検出なし |
| `エアーギャップ`（音引きあり）の混入 | ✅ 検出なし |
| `VSCode`・`vscode`（本文中）の混入 | ✅ 検出なし（タグ・パス内は許容） |
| 外部 URL の本文混入（参考リンクセクション外） | ✅ 検出なし |
| `air-gap` の本文混入（YAML タグ以外） | ✅ 検出なし |
| 章間ナビゲーションリンクの正確性 | ✅ 全リンク正常 |
| コードブロックの言語識別子 | ✅ 全ブロックに付与済み |
| 本文中の H1 見出し混入 | ✅ 検出なし（全章 H2 始まり） |
| フロントマター共通タグ（continue/vscode/on-premise/air-gap） | ✅ 全ファイルに存在 |

---

## 5. 修正優先順位

1. **H-1**（`api_base` vs `apiBase`）: 10 ファイルに影響、読者がコードをコピーすると接続エラーが発生する最重要課題
2. **H-2**（VS Code バージョン不一致）: 第 1・2 章の冒頭で矛盾が生じ、読者の信頼を損なう
3. **M-1**（フロントマタータイトル）: 5 ファイルの単純置換で対応可能
4. **M-5**（画像欠落）: ビルド結果に視覚的な欠損が生じる
5. **M-2**（Rules パス誤記）: 読者が誤った場所にファイルを配置するリスク
6. **M-3**（埋め込みモデルスキーマ不一致）: 設定ミスにつながる
7. **M-4**（mcpServers スキーマ不一致）: 設定ミスにつながる
8. **L-1**（リンクテキスト表記揺れ）: 機能には影響なし

---

## 6. 備考

- 本レポートは `REVIEW_INSTRUCTIONS.md`（チェック項目 3.1〜3.10）に基づき実施した。
- エアギャップ適合性（3.1）については、全インターネット接続操作が `!!! warning` アドモニションで適切に分離されており、問題は検出されなかった。
- 本レポートを受けて `PROGRESS.md` のセクション 4・5 を更新済み。
