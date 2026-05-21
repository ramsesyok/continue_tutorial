---
title: "第12章 MCP サーバの社内利用"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - mcp
  - advanced
---

## この章で学ぶこと

- <glossary:MCP>（Model Context Protocol）の概要と、Continue における位置づけを理解する
- <glossary:エアギャップ>環境で利用可能な <glossary:MCPサーバ>の種類と選定基準を把握する
- ファイルシステム・社内 Git・ローカル DB の 3 つのサーバを実際にセットアップできる
- `config.yaml` の `mcpServers` セクションに接続設定を記述し、動作確認できる
- MCP サーバを社内導入する際のセキュリティ上の留意点を理解する

---

## MCP とは何か

MCP（Model Context Protocol）とは、<glossary:LLM>（大規模言語モデル）がファイル操作・データベース参照・コマンド実行といった外部ツールやデータソースと標準化された方法でやり取りするためのプロトコルです。2024 年に Anthropic が策定・公開し、Continue を含む複数の AI コーディングツールが対応しています。

MCP を使わない場合、LLM はテキストの生成しかできません。しかし MCP サーバを経由することで、LLM は「ファイルを読む」「Git ログを取得する」「SQL をクエリする」といったアクションを実際に実行できるようになります。MCP はこれらの操作を統一したインターフェースで抽象化しており、ツールの実装詳細を LLM 側が知る必要がありません。

!!! note "MCP はオープン仕様"
    MCP の仕様は公開されており、誰でも独自のサーバを実装できます。公式・コミュニティ製のサーバが多数存在しますが、エアギャップ環境では外部通信を必要としないものだけを選ぶ必要があります。

### Continue と MCP の関係

Continue では、第 9 章で扱った **<glossary:Agent> 機能**が MCP と最も密接に連携します。Agent モードでは、Continue が LLM に「利用可能なツール一覧」を提示し、LLM がツールの呼び出し要求を出すと Continue がその要求を MCP サーバに転送して実行結果を受け取り、次の LLM <glossary:推論>に渡します。

```
ユーザー指示
    ↓
Continue（Agent モード）
    ↓ ツール一覧を提示
ローカル LLM
    ↓ ツール呼び出し要求
Continue
    ↓ リクエスト転送
MCP サーバ（ローカル）
    ↓ 実行結果を返却
Continue → LLM → 回答生成
```

<glossary:Chat> モード・<glossary:Edit> モードでは <glossary:MCPツール>は呼び出されません。MCP を活用したい場合は必ず **Agent モード**（`⌘+.` / `Ctrl+.`）を使ってください。

---

## エアギャップ環境での MCP 利用方針

MCP サーバには大きく 2 種類があります。

| 種別 | 説明 | エアギャップ可否 |
| --- | --- | --- |
| ローカル操作型 | ファイルシステム・ローカル DB・社内 Git など、ネットワーク内リソースを操作する | **利用可** |
| 外部 API 型 | GitHub.com・Slack・Google Drive など、インターネット上の SaaS に接続する | **利用不可** |

エアギャップ環境では**ローカル操作型のみ**を使用します。サーバを選定・導入する際は以下の基準で判断してください。

- 起動時・実行時にインターネットへの通信が発生しないか
- 依存パッケージをすべてオフラインで調達できるか
- 設定ファイル内に外部 URL のハードコードがないか

!!! warning "外部 API 型サーバの注意"
    コミュニティ製 MCP サーバの中には、設定次第でローカルとクラウドの両方に対応するものがあります。エアギャップ環境では、接続先 URL を社内<glossary:エンドポイント>に固定し、外部 URL を設定しないよう徹底してください。

---

## 事前準備：ランタイムのオフライン配布

多くの MCP サーバは **Node.js** または **Python** で実装されています。エアギャップ環境でこれらを動かすには、ランタイムと依存パッケージを事前にオフラインで配布しておく必要があります。

### Node.js パッケージのオフライン配布

インターネットに接続できる作業 PC 上で、次のコマンドを実行してパッケージをアーカイブします。

```bash
# 例: @modelcontextprotocol/server-filesystem のアーカイブを作成
npm pack @modelcontextprotocol/server-filesystem
# → modelcontextprotocol-server-filesystem-X.Y.Z.tgz が生成される
```

生成された `.tgz` ファイルと、依存パッケージ一式を社内配布リポジトリ（Artifactory / Nexus 等）またはファイルサーバに転送します。エアギャップ環境のマシンからは、社内リポジトリ経由でインストールします。

```bash
# 社内 npm レジストリからインストールする場合
npm install --registry http://<your-internal-registry>/ @modelcontextprotocol/server-filesystem

# ローカルアーカイブから直接インストールする場合
npm install ./modelcontextprotocol-server-filesystem-X.Y.Z.tgz
```

### Python パッケージのオフライン配布

```bash
# 作業 PC 上でホイールファイルをダウンロード
pip download mcp -d ./mcp-packages

# 社内マシンへのコピー後、オフラインインストール
pip install --no-index --find-links ./mcp-packages mcp
```

!!! tip "Node.js / Python ランタイム本体のオフライン取得"
    Node.js の場合は公式サイトから `.tar.gz`（Linux）または `.msi`（Windows）をダウンロードして配布します。Python の場合も同様に公式インストーラを配布してください。いずれも社内に一度持ち込んだバージョンを固定して運用することを推奨します。

---

## ローカル MCP サーバの導入例

ここでは社内環境でよく使われる 3 つの MCP サーバを取り上げ、それぞれのセットアップ手順をハンズオン形式で示します。

### ファイルシステム MCP サーバ

`@modelcontextprotocol/server-filesystem` は、指定したディレクトリ以下のファイル読み書き・一覧取得を LLM が直接行えるようにするサーバです。ソースコードや設定ファイルを参照しながらコードを修正する用途に適しています。

**インストール**

```bash
# 社内リポジトリ経由でグローバルインストール
npm install -g --registry http://<your-internal-registry>/ @modelcontextprotocol/server-filesystem
```

**動作確認（手動起動）**

```bash
# 許可するディレクトリを引数で渡して起動
npx @modelcontextprotocol/server-filesystem /home/user/projects
```

起動後、標準入力に JSON-RPC メッセージを送ることでツール呼び出しを確認できます。ただし通常は Continue が自動で起動・終了を管理するため、手動起動は疎通確認のみに使います。

**公開するディレクトリの選定**

ファイルシステムサーバに渡すパスは、LLM がアクセスしてよい範囲に限定してください。`/` や `C:\` 全体を指定することは避け、プロジェクトルート等の必要最小限のディレクトリのみを指定します。

### 社内 Git リポジトリ連携

社内 GitLab / Gitea などに接続してコミット履歴・差分・ブランチ情報を取得できる MCP サーバを使うと、「このコミットで何が変わったか」「過去のレビューコメントは何か」といった質問に LLM が直接回答できるようになります。

ここでは `@modelcontextprotocol/server-git`（公式実装）を例に示します。

**インストール**

```bash
npm install -g --registry http://<your-internal-registry>/ @modelcontextprotocol/server-git
```

**設定のポイント**

接続先は社内 Git サーバのエンドポイントのみを指定します。`config.yaml` での設定例は後述の「Continue への接続設定」を参照してください。

```yaml
# config.yaml 内の記述例（抜粋）
mcpServers:
  - name: "git"
    command: "npx"
    args:
      - "@modelcontextprotocol/server-git"
      - "--repository"
      - "/home/user/projects/myapp"
```

!!! warning "外部 Git ホストへの接続禁止"
    `--remote` オプションや環境変数で `github.com` / `gitlab.com` 等の外部 URL を指定しないよう注意してください。エアギャップ環境では社内 Git サーバの URL のみ使用します。

### SQLite／ローカル DB 参照

社内に SQLite や PostgreSQL（ローカル稼働）などのデータベースがある場合、`@modelcontextprotocol/server-sqlite` を使うことで LLM がスキーマを確認したりクエリを発行したりできるようになります。

**インストール**

```bash
npm install -g --registry http://<your-internal-registry>/ @modelcontextprotocol/server-sqlite
```

**起動引数の例**

```bash
# 対象 DB ファイルをパスで指定
npx @modelcontextprotocol/server-sqlite /var/data/internal.db
```

**ユースケース例**

- 「`orders` テーブルのカラム定義を教えて」→ LLM がスキーマを取得して回答
- 「先月の売上合計を SQL で出して」→ LLM が SQL を生成 → MCP サーバが実行 → 結果を回答

!!! warning "書き込み操作の取り扱い"
    MCP サーバによっては `INSERT` / `UPDATE` / `DELETE` も実行できます。意図しないデータ変更を防ぐために、**読み取り専用ユーザー**でサーバを起動することを強く推奨します。

---

## Continue への接続設定

MCP サーバの準備ができたら、`config.yaml` に接続設定を記述します。設定ファイルは VS Code 上で<glossary:コマンドパレット>（`⌘+Shift+P` / `Ctrl+Shift+P`）→「Continue: Open <glossary:config.yaml>」から開けます。

### stdio 接続（推奨）

ほとんどのローカル MCP サーバは **stdio**（標準入出力）経由で通信します。Continue がサーバプロセスを起動・終了を管理するため、設定が最もシンプルです。

```yaml
mcpServers:
  - name: "filesystem"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-filesystem"
      - "/home/user/projects"
  - name: "git"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-git"
      - "--repository"
      - "/home/user/projects/myapp"
  - name: "sqlite"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-sqlite"
      - "/var/data/internal.db"
```

各フィールドの意味は次の通りです。

| フィールド | 説明 |
| --- | --- |
| `name` | Continue 上でのサーバの識別名。ツール一覧に表示される |
| `command` | サーバを起動するコマンド（`npx`、`node`、`python` 等） |
| `args` | コマンドに渡す引数の配列 |
| `env`（省略可） | サーバプロセスに渡す環境変数のキーバリュー |

### SSE（HTTP）接続

すでに常駐プロセスとして社内サーバで稼働している MCP サーバに接続する場合は、SSE（Server-Sent Events）接続を使います。

```yaml
mcpServers:
  - name: "internal-tools"
    url: "http://<your-mcp-server-host>:8080/sse"
```

!!! note "本構成では外部 URL は使用しません"
    `url` フィールドに指定するのは必ず社内ネットワーク内のエンドポイントにしてください。インターネット上の URL を指定するとエアギャップ前提に反します。

### 動作確認の手順

設定を保存したら、次の手順で MCP サーバが正しく認識されているか確認します。

1. VS Code を再起動するか、コマンドパレットから「Continue: Reload Config」を実行する
2. Continue の Chat パネルを開き、右下の「ツール」アイコン（🔧）をクリックする
3. 設定した MCP サーバ名と、そのサーバが提供するツール一覧が表示されることを確認する
4. Agent モード（`⌘+.` / `Ctrl+.`）で「プロジェクトルートのファイル一覧を教えて」と入力し、LLM がファイルシステムサーバを呼び出すことを確認する

**よくある接続エラーと対処法**

| 症状 | 原因の可能性 | 対処 |
| --- | --- | --- |
| ツール一覧にサーバが表示されない | `command` のパスが通っていない | `which npx` / `where npx` でパスを確認し、フルパスで指定する |
| 「Server disconnected」エラー | 依存パッケージが不足している | `npm install` のログを確認し、依存パッケージを再インストールする |
| ツールが呼ばれるが結果が空 | 引数（パスや URL）が誤っている | `args` の値を見直し、手動起動で疎通を確認する |

---

## セキュリティ上の留意点

MCP サーバはローカルで動作するとはいえ、LLM がファイルを読み書きしたり、データベースにクエリを発行したりできる強力な機能です。社内導入にあたっては、以下のセキュリティ観点を必ず確認してください。

### ツール承認モデルと最小権限の原則

Continue の Agent モードでは、LLM がツール呼び出しを要求するたびに、ユーザーに承認を求めるダイアログが表示されます（デフォルト設定）。この承認ステップを無効化すると、LLM が意図しないファイル変更やクエリを自動実行してしまう危険があります。

```yaml
# config.yaml でツール承認を明示的に有効化する（デフォルト値だが明記を推奨）
agent:
  autoApproveTools: false
```

MCP サーバ自体の実行ユーザーについても、<glossary:最小権限の原則>を適用してください。

- ファイルシステムサーバ：読み取り専用でよい場合は `chmod 444` または読み取り専用マウントで制限する
- Git サーバ：`fetch` / `log` のみ許可し、`push` 権限を持つ認証情報を渡さない
- DB サーバ：`SELECT` のみ許可する読み取り専用ユーザーで接続する

### ログと監査

MCP サーバのリクエスト・レスポンスをログに記録しておくと、「LLM が何を読み取り、何を実行したか」を後から検証できます。

Continue 側のログは次のパスに出力されます。

```text
# macOS / Linux
~/.continue/logs/core.log

# Windows
%USERPROFILE%\.continue\logs\core.log
```

MCP サーバ側のログは、各サーバの実装によって異なります。Node.js 製サーバの場合は、起動コマンドの前後でリダイレクトを使いログファイルに保存できます。

```bash
# ログをファイルに出力しながら起動する例
npx @modelcontextprotocol/server-filesystem /home/user/projects >> /var/log/mcp-filesystem.log 2>&1
```

ログは定期的に確認し、不審なパスへのアクセスや大量のクエリが発生していないか監査することを推奨します。

### 悪意あるツール呼び出しへの対処

<glossary:プロンプト>インジェクション（ユーザーやファイルの内容に埋め込まれた悪意ある指示）により、LLM が意図しないツールを呼び出す可能性があります。以下の対策を組み合わせてリスクを低減してください。

- **承認ダイアログを必ず確認する**：ツール呼び出しの内容（対象ファイル・クエリ内容）を承認前に目視で確認する習慣をつける
- **アクセス範囲を限定する**：ファイルシステムサーバに渡すパスを必要最小限にする
- **不要なツールは無効化する**：`config.yaml` の `mcpServers` に登録するサーバは業務上必要なものだけにとどめる
- **定期的なログレビュー**：異常なアクセスパターンを早期に検知する

!!! warning "信頼できないコードベースでの Agent 利用"
    外部から取得したコードや、内容が十分に把握できていないファイルをコンテキストに含める場合は、MCP ツールが予期しない動作をする可能性があります。不審な承認リクエストが表示された場合は「拒否」を選択し、LLM の応答を再確認してください。

---

## まとめ

- MCP（Model Context Protocol）は LLM と外部ツールをつなぐ標準プロトコルであり、Continue の Agent モードから利用できる
- エアギャップ環境では、外部 API に依存しない**ローカル操作型**の MCP サーバのみを使用する
- Node.js / Python ランタイムと依存パッケージは事前に社内配布しておく必要がある
- `config.yaml` の `mcpServers` セクションに `command` / `args` を記述するだけで stdio 接続が可能
- ツール承認は必ず有効にし、最小権限・ログ監査・アクセス範囲限定によってリスクを管理する

---

## 次の章へ

次は [第 13 章 セキュリティと運用](13-security-operations.md) で、機密コードの取り扱いポリシー・ログ運用・ユーザー教育など、組織全体での Continue 運用を体系的に扱います。

---

## 参考リンク

- [Model Context Protocol 公式仕様](https://spec.modelcontextprotocol.io/)
- [MCP サーバ公式リポジトリ（modelcontextprotocol/servers）](https://github.com/modelcontextprotocol/servers)
- [Continue 公式ドキュメント - MCP](https://docs.continue.dev/features/mcp)
- [MkDocs Material - Admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/)
