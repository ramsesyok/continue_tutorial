---
title: "第 14 章 トラブルシューティング"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - troubleshooting
  - logs
---

## この章で学ぶこと

- Continue 拡張・VS Code・ローカル LLM サーバのログの場所と基本的な読み方
- 症状ごとの切り分け手順（接続不可・補完停止・応答エラー・インデックス異常・Edit 不具合）
- レイテンシ増大や VS Code の重さを引き起こすパフォーマンス問題の特定方法
- 補完や回答の品質が低いときの調整ポイント
- それでも解決しないときの環境リセット・再インストール手順

---

## ログの場所と読み方

問題が発生したときは、まずログを確認することが解決への最短経路です。Continue を取り巻くログは「Continue 拡張のログ」「VS Code 本体のログ」「ローカル LLM サーバのログ」の 3 層構造になっています。原則として **上から順に確認し、エラーが見つかった層に絞って深掘り** してください。

### Continue 拡張のログを開く

VS Code の上部メニューから **「表示（View）」→「出力（Output）」** を選択し、右上のドロップダウンで **「Continue」** を選択します。ここには Continue が LLM に送ったリクエスト・受け取ったレスポンス・内部エラーがリアルタイムに流れます。

```text
# 出力パネルで確認できる主な情報
[INFO]  Sending request to http://<your-internal-llm-host>:11434/api/chat
[ERROR] Connection refused: http://<your-internal-llm-host>:11434
[WARN]  Context length exceeded. Truncating to 4096 tokens.
```

!!! tip "ログレベルを上げる"
    詳細なデバッグ情報が必要な場合は、`config.yaml` に次の行を追加してください。VS Code を再起動すると、出力パネルのログが詳細になります。

    ```yaml
    # config.yaml（抜粋）
    requestOptions:
      # 開発・調査時のみ有効にする。本番運用では false に戻す
      verbose: true
    ```

### VS Code 本体のログ

Continue 拡張がクラッシュする・パネルが開かない・コマンドが無反応、といった場合は VS Code の開発者コンソールを確認します。

1. メニューの **「ヘルプ（Help）」→「開発者ツールを切り替える（Toggle Developer Tools）」** をクリックします
2. 「Console」タブを開き、赤字のエラーを探します
3. `Extension Host` から始まるスタックトレースがあれば、拡張機能側の問題です

!!! note "開発者ツールはデバッグ専用"
    開発者ツールを開いたまま通常作業をすると VS Code のパフォーマンスに影響します。調査が終わったら必ず閉じてください。

### ローカル LLM サーバのログ

ローカル LLM サーバのログは、サーバ種別によって場所が異なります。

**Ollama** の場合

```bash
# ターミナルで起動している場合は標準出力にそのまま表示される
# サービスとして動いている場合は journald または専用ログを確認する
journalctl -u ollama -f
```

**vLLM** の場合

```bash
# 起動コマンドの標準出力にログが流れる
# systemd サービス化している場合
journalctl -u vllm -f
```

**LM Studio** の場合

LM Studio のサーバ画面下部の「Server Logs」タブでリアルタイムのリクエストログを確認できます。

---

## 症状別トラブルシューティング

以下のセクションでは、実際に発生しやすい症状ごとに切り分け手順を示します。各症状の先頭に示すチェックリストを **上から順番に** 確認してください。

### LLM に接続できない

**症状の例**

- Chat パネルに「Connection refused」「ECONNREFUSED」と表示される
- Autocomplete が一切反応しない
- Continue の出力パネルに `[ERROR] Failed to connect` が流れる

**切り分け手順**

**ステップ 1: LLM サーバが起動しているか確認する**

```bash
# Ollama の例
curl http://<your-internal-llm-host>:11434/api/tags
# vLLM の例
curl http://<your-internal-llm-host>:8000/v1/models
```

正常であれば JSON のモデル一覧が返ります。`curl: (7) Failed to connect` が出た場合は、サーバが停止しているか、ホスト名・ポートが誤っています。

**ステップ 2: `config.yaml` のエンドポイントを確認する**

```yaml
# config.yaml（例: Ollama 接続）
models:
  - name: local-chat
    provider: ollama
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"  # ← 末尾スラッシュは不要
```

よくあるミスは「ポート番号の誤り」「`http://` の記載漏れ」「末尾に不要なパスを追加している」などです。

**ステップ 3: ファイアウォール・プロキシを確認する**

エアギャップ環境では LAN 内のファイアウォールが LLM サーバへの通信を遮断している場合があります。ネットワーク管理者に当該ポートの通信許可を確認してください。

**ステップ 4: VS Code を再起動する**

設定変更後は必ず VS Code を完全に再起動してください。リロード（`Ctrl+Shift+P` → 「Developer: Reload Window」）でも反映されますが、確実を期す場合はプロセスごと終了・起動します。

---

### Autocomplete が動作しない・候補が出ない

**症状の例**

- Tab キーを押しても補完候補が表示されない
- 補完がごくまれにしか出ない
- 補完の候補が意図と全く合っていない

**切り分け手順**

**ステップ 1: Autocomplete 用モデルが設定されているか確認する**

Autocomplete（タブ補完）は Chat モデルとは別の `tabAutocompleteModel` を使います。設定がない場合、Autocomplete は動作しません。

```yaml
# config.yaml（例）
tabAutocompleteModel:
  name: local-autocomplete
  provider: ollama
  model: <your-autocomplete-model-name>    # 例: qwen2.5-coder:1.5b
  apiBase: "http://<your-internal-llm-host>:11434"
```

!!! warning "Chat モデルをそのまま使わない"
    パラメータ数の大きい Chat モデルを Autocomplete に使うと、レイテンシが高くなりすぎて補完が機能しなくなります。1B〜7B 程度のコード特化モデルを別途用意することを推奨します（詳細は [第 7 章](07-autocomplete.md) を参照）。

**ステップ 2: タイムアウトを確認する**

LLM サーバの応答が遅い場合、補完が内部タイムアウトで破棄されています。Continue の出力パネルに `Autocomplete timed out` が流れていないか確認してください。対処として `debounceDelay` を大きくするか、より軽量なモデルに切り替えます。

```yaml
# config.yaml（抜粋）
tabAutocompleteOptions:
  debounceDelay: 500         # 既定値 300ms → 遅い環境では 500〜1000 に増やす
  maxPromptTokens: 1024      # コンテキストを削減してレイテンシを改善する
```

**ステップ 3: 対象ファイルの言語が対応しているか確認する**

Autocomplete が対応していない言語の拡張子のファイルでは候補が出ません。Continue の公式ドキュメントでサポート言語を確認してください。

---

### Chat・Agent が応答しない、またはエラーになる

**症状の例**

- Chat パネルに送信したメッセージが永遠に処理中のまま止まる
- 「context length exceeded」「out of memory」などのエラーが返る
- Agent がツールを呼び出したまま固まる

**切り分け手順**

**ステップ 1: コンテキスト長超過を確認する**

長いファイルや多くの `@file` を添付した場合、モデルのコンテキスト上限を超えてエラーになります。Continue の出力パネルで `Context length exceeded` を確認し、添付するファイルの数を減らすか、`maxPromptTokens` を設定してください。

```yaml
# config.yaml（抜粋）
models:
  - name: local-chat
    provider: ollama
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    requestOptions:
      maxTokens: 2048        # 生成トークン上限
```

**ステップ 2: LLM サーバの Out-of-Memory を確認する**

GPU メモリ不足でサーバ側がクラッシュしている可能性があります。LLM サーバのログ（前述「ローカル LLM サーバのログ」参照）で `CUDA out of memory` や `OOM` が出ていないか確認してください。より小さいモデルへの切り替え、または量子化レベルの変更（例: Q8 → Q4）が有効です。

**ステップ 3: ストリーミング接続を確認する**

一部のネットワーク環境では Server-Sent Events（SSE）によるストリーミング通信がプロキシでブロックされることがあります。Continue の出力パネルに `stream ended unexpectedly` が出る場合、ネットワーク管理者に SSE の通信許可を確認してください。

下表に主なエラーメッセージと原因・対処をまとめます。

| エラーメッセージ（抜粋） | 主な原因 | 対処 |
| --- | --- | --- |
| `Connection refused` | サーバ未起動 / ポート誤り | サーバ起動確認・`apiBase` を修正 |
| `context length exceeded` | コンテキスト超過 | 添付ファイル削減・`maxPromptTokens` を設定 |
| `CUDA out of memory` | GPU メモリ不足 | 小型モデルへ変更・量子化レベルを下げる |
| `stream ended unexpectedly` | SSE ブロック | プロキシ設定を確認 |
| `model not found` | モデル名の誤り | サーバ上のモデル名を `curl` で確認 |

---

### コードベースインデックスが更新されない

**症状の例**

- `@codebase` に聞いても古いコードの内容が返る
- Continue のステータスバーに「Indexing…」が消えない
- インデックスが途中で止まっているように見える

**切り分け手順**

**ステップ 1: Embedding モデルの設定を確認する**

コードベースのインデックス作成にはローカルの Embedding モデルが必要です。設定がない場合、インデックスは生成されません。

```yaml
# config.yaml（例）
embeddingsProvider:
  provider: ollama
  model: <your-embedding-model-name>    # 例: nomic-embed-text
  apiBase: "http://<your-internal-llm-host>:11434"
```

!!! warning "本構成では外部 Embedding モデルは使用しません"
    クラウドの Embedding API（OpenAI Embeddings 等）はエアギャップ環境では利用できません。必ずローカルで動作する Embedding モデルを設定してください（詳細は [第 10 章](10-internal-models-context.md) を参照）。

**ステップ 2: インデックスキャッシュを削除して再構築する**

インデックスのキャッシュファイルが壊れている場合、削除して再構築します。

```bash
# Windows の場合（PowerShell）
Remove-Item -Recurse -Force "$env:USERPROFILE\.continue\index"

# macOS / Linux の場合
rm -rf ~/.continue/index
```

削除後、VS Code を再起動すると自動的にインデックスが再構築されます。

**ステップ 3: `.continueignore` を確認する**

`.gitignore` に近い書式の `.continueignore` ファイルをリポジトリルートに置くと、特定のファイル・ディレクトリをインデックスから除外できます。意図せず重要なディレクトリが除外されていないか確認してください。

```text
# .continueignore の例
node_modules/
dist/
*.min.js
```

---

### Edit の差分が正しく適用されない

**症状の例**

- Edit を実行したが差分が何も表示されない
- 差分が表示されたが、適用するとコードが壊れる
- インライン diff が延々とローディング中になる

**切り分け手順**

**ステップ 1: Edit に使用するモデルを確認する**

Edit 機能は Chat モデルを使いますが、コード変換の精度が低いモデルでは不正な差分が生成されます。コード特化モデル（Qwen Coder 系、DeepSeek Coder 系など）への変更を検討してください。

**ステップ 2: 選択範囲が適切か確認する**

Edit（`Ctrl+I`）を使うときは、変更したいコードを事前に選択してから起動します。選択なしで呼び出すと、カーソル行周辺のみが対象になるため、意図した範囲に変更が適用されないことがあります。

**ステップ 3: コンテキスト長を確認する**

ファイル全体が長い場合、Edit のコンテキスト上限を超えて差分生成に失敗します。ファイルを分割するか、編集対象の関数・クラスだけを選択して Edit を実行してください。

---

## パフォーマンス問題の切り分け

### レイテンシが高い

Chat の応答や Autocomplete の候補表示が遅い場合、原因を次の順に絞り込みます。

**1. ネットワーク遅延を確認する**

```bash
# LLM サーバへの ping で基本的なネットワーク遅延を測定する
ping <your-internal-llm-host>
```

往復遅延（RTT）が 10ms を大きく超える場合、ネットワーク経路に問題がある可能性があります。

**2. LLM サーバの GPU 負荷を確認する**

```bash
# サーバ側で GPU 使用率を確認する
nvidia-smi -l 1
```

GPU 使用率が常時 100% に張り付いている場合、他のプロセスとリソースを競合しているか、モデルがサイズオーバーです。

**3. コンテキスト長を削減する**

送信するコンテキストが長いほど処理時間は増加します。以下の設定でコンテキスト上限を制限してください。

```yaml
# config.yaml（抜粋）
tabAutocompleteOptions:
  maxPromptTokens: 1024     # Autocomplete のコンテキスト上限
```

**4. モデルの量子化レベルを見直す**

Q8 より Q4 の方が推論速度が速い傾向があります。品質と速度のバランスを確認しながら量子化レベルを調整してください。

### VS Code が重くなる・メモリを大量消費する

**インデックスサイズの確認**

大規模リポジトリをすべてインデックスしている場合、VS Code のメモリ使用量が大きくなります。`.continueignore` でビルド成果物・依存パッケージ等を除外してください。

```text
# .continueignore（大規模リポジトリの推奨設定例）
node_modules/
.venv/
__pycache__/
dist/
build/
*.lock
```

**ファイル監視の絞り込み**

VS Code 自体のファイル監視（`files.watcherExclude`）の設定と組み合わせることで、システム全体のリソース消費を抑えられます。VS Code の `settings.json` に次を追加してください。

```json
{
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/.venv/**": true,
    "**/dist/**": true
  }
}
```

---

## 品質チューニング

### モデルの選定を見直す

Continue の各機能には、それぞれ適したモデルのサイズ・種類があります。

| 機能 | 推奨モデル規模 | 備考 |
| --- | --- | --- |
| Chat | 7B〜34B | コード理解力と日本語対応を優先 |
| Autocomplete | 1B〜7B | 応答速度を最優先。コード特化モデルを選ぶ |
| Edit | 7B〜34B | 差分生成精度が重要。Chat と同モデルで可 |
| Embedding | 専用モデル | nomic-embed-text 等の Embedding 専用モデルを使う |

!!! tip "まず小さいモデルで試す"
    大きいモデルが必ずしも体験が良いとは限りません。Autocomplete はとくに応答速度が体験を左右するため、1B 程度の小型モデルから試して速度を確認してから、徐々にサイズを上げる方法が効率的です。

### システムプロンプトと Rules の調整

Chat や Edit の回答スタイルが期待と異なる場合、システムプロンプトや Rules で出力をコントロールできます。

```yaml
# config.yaml（例: システムプロンプトの設定）
models:
  - name: local-chat
    provider: ollama
    model: <your-model-name>
    apiBase: "http://<your-internal-llm-host>:11434"
    systemMessage: |
      あなたは社内開発チームを支援するコーディングアシスタントです。
      回答は日本語で行い、コードには必ずコメントを日本語で付けてください。
```

Rules ファイル（`.continue/rules/`）を使うと、コーディング規約や命名規則を Continue に指示できます。詳細は [第 11 章](11-customization.md) を参照してください。

---

## 環境リセットと再インストール手順

上記のすべての手順を試しても問題が解決しない場合、Continue の設定とキャッシュを完全にリセットすることを検討してください。

!!! warning "リセット前に設定をバックアップする"
    リセットすると `config.yaml` を含む設定ファイルが失われます。必ず事前にバックアップを取ってください。

**ステップ 1: Continue の設定・キャッシュをバックアップする**

```bash
# Windows（PowerShell）
Copy-Item -Recurse "$env:USERPROFILE\.continue" "$env:USERPROFILE\.continue_backup"

# macOS / Linux
cp -r ~/.continue ~/.continue_backup
```

**ステップ 2: Continue の設定・キャッシュを削除する**

```bash
# Windows（PowerShell）
Remove-Item -Recurse -Force "$env:USERPROFILE\.continue"

# macOS / Linux
rm -rf ~/.continue
```

**ステップ 3: Continue 拡張をアンインストールする**

VS Code の拡張機能パネルで Continue を右クリックし、「アンインストール」を選択します。VS Code を再起動して拡張が完全に削除されたことを確認してください。

**ステップ 4: VSIX ファイルから再インストールする**

エアギャップ環境では Marketplace から直接インストールできないため、事前にオフライン配布された VSIX ファイルを使います。VSIX の入手場所は社内の配布リポジトリを確認してください。

```bash
# コマンドラインからインストールする場合
code --install-extension /path/to/continue-<version>.vsix
```

詳細なインストール手順は [第 3 章](03-offline-install.md) を参照してください。

**ステップ 5: 設定を復元する**

バックアップから `config.yaml` だけをコピーして動作を確認してください。一度にすべてを復元すると、問題のある設定も一緒に戻る可能性があります。

```bash
# Windows（PowerShell）
Copy-Item "$env:USERPROFILE\.continue_backup\config.yaml" "$env:USERPROFILE\.continue\config.yaml"

# macOS / Linux
cp ~/.continue_backup/config.yaml ~/.continue/config.yaml
```

---

## まとめ

- Continue のトラブルシューティングは「Continue ログ → VS Code ログ → LLM サーバログ」の 3 層を上から順に確認することが基本です
- 接続問題は `curl` による疎通確認と `config.yaml` のエンドポイント設定確認で大半が解決します
- Autocomplete の無反応はタイムアウトかモデル未設定が原因であることが多く、`tabAutocompleteModel` の設定と `debounceDelay` の調整が有効です
- パフォーマンスが悪い場合は、コンテキスト長の削減・軽量モデルへの変更・`.continueignore` によるインデックス対象の絞り込みを試してください
- すべて試して解決しない場合は、設定をバックアップしてから環境をリセットし、VSIX で再インストールします

## 次の章へ

これで本チュートリアルの全章が完了です。各設定オプションの詳細は [付録 A config.yaml リファレンス](appendix/config-reference.md) を、よく使うショートカットの一覧は [付録 B キーボードショートカット](appendix/shortcuts.md) を参照してください。

## 参考リンク

- [Continue 公式ドキュメント](https://docs.continue.dev/)
- [Continue GitHub Issues（既知の問題の確認）](https://github.com/continuedev/continue/issues)
- [Material for MkDocs – admonition](https://squidfunk.github.io/mkdocs-material/reference/admonitions/)
