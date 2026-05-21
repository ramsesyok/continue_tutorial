---
title: "第13章 セキュリティと運用"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - security
  - operations
---

## この章で学ぶこと

- Continue を使う際のデータフローを理解し、<glossary:エアギャップ>環境での安全性の根拠を把握する
- 機密コードを誤って <glossary:LLM> に送信しないための設定・ポリシーを整備する
- Continue が出力するログの種類・場所・管理方法を習得する
- VS Code 拡張と <glossary:config.yaml> の配布・更新を組織的に運用する手順を学ぶ
- ユーザー教育と利用ガイドラインの整備によって、組織全体でのリスクを低減する

---

## Continue を使う際のセキュリティモデル

Continue は、ユーザーが入力した<glossary:プロンプト>や選択したコードを LLM に送信することで動作します。ここでは「何がどこに送られるのか」を整理し、エアギャップ環境でのセキュリティ上の根拠を確認します。

### データフローの全体像

Continue は VS Code の拡張として動作し、以下の経路でデータを送受信します。

```text
VS Code 上のコード・プロンプト
        ↓
Continue 拡張（ローカルプロセス）
        ↓
ローカル LLM エンドポイント（例: http://localhost:11434）
        ↓
LLM による推論結果
        ↓
Continue 拡張 → VS Code 上に表示
```

ローカル LLM（<glossary:Ollama>・<glossary:vLLM>・<glossary:LMStudio> など）を使用している場合、コードとプロンプトはすべてイントラネット内で完結します。インターネット上の外部 API には一切送信されません。

!!! note "エアギャップ環境における確認済みの前提"
    本チュートリアルは「Continue からローカル LLM への接続設定が完了しており、外部通信が遮断されている」ことを前提にしています。外部通信の遮断確認については [第 4 章](04-telemetry-airgap-verification.md) を参照してください。

### 送信されるデータの種類

LLM に送信される情報は、主に以下の 3 種類です。

- **プロンプトテキスト**: ユーザーがチャットや <glossary:Edit> で入力したテキスト
- **コードコンテキスト**: 選択したコード、`@file` で指定したファイル、`@codebase` による検索結果など
- **システムプロンプト**: Rules（フォルダ内の `.continue/rules/` 配下のファイル）の内容

いずれもローカル LLM <glossary:エンドポイント>にのみ送信されます。ただし、**contextLength の設定値まで自動的にコードが含まれる**点に注意が必要です。次節でこの点を詳しく説明します。

---

## 機密コードの取り扱いポリシー

エアギャップ環境でも、ローカル LLM サーバを共有している場合や、LLM のプロンプトログが記録される環境では、機密情報が意図せず記録・参照されるリスクがあります。組織としてのポリシーを整備し、設定で補強することが重要です。

### 送信範囲を理解する

Continue は、ユーザーが明示的に選択したコード以外にも、周辺コードを自動的にコンテキストとして LLM に送ります。この範囲は `config.yaml` の `defaultCompletionOptions.contextLength` で制御されます。

```yaml
# config.yaml の例（Ollama 接続）
models:
  - name: my-local-llama
    provider: ollama
    model: <your-model-name>
    apiBase: http://<your-internal-llm-host>:11434
    defaultCompletionOptions:
      contextLength: 4096   # ← この値のトークン数まで周辺コードが含まれる
```

`contextLength` を大きくすると補完精度は向上しますが、その分だけ多くのコードが LLM に渡ります。機密性の高いリポジトリでは、**必要最小限の値（2048〜4096 程度）に抑える**ことを推奨します。

!!! warning "Autocomplete のコンテキストについて"
    <glossary:Autocomplete>（タブ補完）では、カーソル前後のコードが自動的にコンテキストとして使われます。機密性の高いファイルを編集する際は、Autocomplete を一時的に無効化することも検討してください（`Ctrl+Shift+P` → `Continue: Toggle Autocomplete`）。

### .continueignore でファイルを除外する

`.continueignore` ファイルを使うと、特定のファイルやディレクトリを Continue の補完・<glossary:インデックス>・コンテキスト検索から除外できます。書式は `.gitignore` と同一です。

`.continueignore` はリポジトリルートまたはユーザーのホームディレクトリ（`~/.continue/.continueignore`）に配置します。

```gitignore
# .continueignore の例

# 秘密鍵・証明書
*.pem
*.key
secrets/

# 個人情報を含むファイル
**/personal_data/
**/pii/

# ライセンス制約のある外部ライブラリのソースコード
vendor/proprietary/

# 環境変数ファイル
.env
.env.*
```

!!! tip "リポジトリ共通設定として配布する"
    `.continueignore` をリポジトリに含めて Git 管理することで、チーム全員に同じ除外ルールを適用できます。組織の社内 Git サーバを通じて配布するのがおすすめです。

### 入力してよいコード・してはいけないコードの指針

組織ごとの状況に応じて、利用ガイドラインに以下のような記載を加えることを推奨します。次の表はたたき台として参照してください。

| 分類 | 例 | Continue への入力 |
| --- | --- | --- |
| 一般ロジック・アルゴリズム | ソートアルゴリズム、ユーティリティ関数 | ✅ 可 |
| 社内 API クライアントコード | 内製ライブラリの実装 | ✅ 可（ポリシー確認の上） |
| 認証情報・秘密鍵 | API キー、パスワード、<glossary:トークン> | ❌ 不可 |
| 個人を特定できる情報（PII） | 氏名・住所・マイナンバーを含むコード | ❌ 不可 |
| 特定顧客との契約に基づく機密コード | NDA 締結プロジェクトのコア実装 | ⚠️ 要確認（法務・情報管理部門に相談） |
| ライセンス制約のある外部コード | 商用ライブラリのソース | ⚠️ 要確認（ライセンス担当に相談） |

---

## ログの場所と管理

### Continue が出力するログの種類

Continue は動作中にさまざまなログを出力します。主なログファイルの場所は以下のとおりです。

| ログの種類 | 場所 | 内容 |
| --- | --- | --- |
| メインログ | `~/.continue/logs/core.log` | Continue コアプロセスの動作ログ（エラー・警告・イベント） |
| デバッグログ | VS Code 出力パネル → `Continue - LLM Prompts/Completions` | LLM に実際に送信されたプロンプトと受信したレスポンス |
| インデックスログ | `~/.continue/index/` 配下 | `@codebase` 向けのコードインデックスキャッシュ |

VS Code の出力パネルは `Ctrl+Shift+U`（macOS: `Cmd+Shift+U`）で開き、右上のドロップダウンから `Continue - LLM Prompts/Completions` を選択すると、送受信内容をリアルタイムで確認できます。

!!! tip "プロンプトログはデバッグに有用"
    補完精度の確認や、誤った応答の原因調査に、プロンプトログは非常に役立ちます。一方で、**プロンプトログには入力したコードがそのまま記録される**ため、取り扱いには注意が必要です。

### ログの保持期間と削除ポリシー

`core.log` はローリング形式で自動的に更新されますが、インデックスキャッシュは明示的に削除しない限り蓄積されます。組織のポリシーに応じて定期的な削除を検討してください。

以下は、90 日以上前のログを削除するスクリプトの例です。OS のタスクスケジューラ（Windows: タスクスケジューラ、Linux/macOS: cron）に登録して定期実行できます。

```bash
#!/bin/bash
# continue-log-cleanup.sh
# ~/.continue/logs/ 配下の 90 日以上前のファイルを削除する

LOG_DIR="$HOME/.continue/logs"
RETENTION_DAYS=90

find "$LOG_DIR" -type f -mtime +"$RETENTION_DAYS" -exec rm -f {} \;
echo "[$(date '+%Y-%m-%d %H:%M:%S')] ログクリーンアップ完了（保持期間: ${RETENTION_DAYS} 日）"
```

```powershell
# continue-log-cleanup.ps1（Windows 用）
# ~/.continue/logs/ 配下の 90 日以上前のファイルを削除する

$LogDir = "$env:USERPROFILE\.continue\logs"
$RetentionDays = 90
$Cutoff = (Get-Date).AddDays(-$RetentionDays)

Get-ChildItem -Path $LogDir -File |
    Where-Object { $_.LastWriteTime -lt $Cutoff } |
    Remove-Item -Force

Write-Host "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') ログクリーンアップ完了（保持期間: $RetentionDays 日）"
```

!!! note "ログ削除前の確認"
    削除前に、組織のセキュリティポリシーやコンプライアンス要件でログの保存期間が定められていないか確認してください。要件がある場合は、削除前にログを安全なストレージへアーカイブしてください。

### 監査証跡としての活用

セキュリティ審査やインシデント対応の場面では、Continue のログが証跡資料として役立つことがあります。以下の手順を参考にしてください。

1. **ログを収集する**: `~/.continue/logs/core.log` と VS Code の出力パネルのログを取得します。出力パネルのログは、パネルを右クリック →「出力のコピー」でクリップボードへコピーできます。
2. **対象期間を絞り込む**: タイムスタンプで対象のイベントを特定します。
3. **提出前に機密情報を除去する**: ログにはプロンプト内容が含まれる場合があります。提出前に個人情報・秘密鍵等が含まれていないか確認し、必要に応じてマスキングを行います。

!!! warning "ログに含まれる情報の取り扱い"
    VS Code の `Continue - LLM Prompts/Completions` ログには、LLM に送信したコードやプロンプトが平文で記録されます。このログファイルを外部に提出する際は、必ず内容を事前に確認してください。

---

## 配布と更新の運用

### VS Code 拡張のバージョン管理

エアギャップ環境では、VS Code Marketplace から自動更新することができません。社内で以下の運用フローを整備することを推奨します。

```text
① 担当者が公式サイトから VSIX ファイルを入手（インターネット環境で実施）
        ↓
② 社内セキュリティチェック（ウイルススキャン、ライセンス確認）
        ↓
③ 社内ファイルサーバ・パッケージリポジトリへの格納
        ↓
④ 各開発者が社内リポジトリから VSIX を取得してインストール
```

特定バージョンを固定して全社配布するには、<glossary:VSIX> ファイルを社内 Web サーバや共有ドライブに置き、バージョンごとにディレクトリを分けて管理する方法が簡単です。

```text
/share/vscode-extensions/
  continue/
    v1.0.0/
      continue-1.0.0.vsix
    v1.1.0/
      continue-1.1.0.vsix
    latest -> v1.1.0/   ← シンボリックリンクで最新バージョンを示す
```

インストール手順については [第 3 章](03-offline-install.md) を参照してください。

### config.yaml の組織展開

Continue の設定ファイル（`config.yaml`）には、個人レベルの設定と組織共有の設定の 2 種類があります。

| スコープ | ファイルの場所 | 用途 |
| --- | --- | --- |
| ユーザー設定 | `~/.continue/config.yaml` | 個人の LLM エンドポイント・モデル選択など |
| <glossary:ワークスペース>設定 | `<リポジトリルート>/.continue/config.yaml` | プロジェクト共通の Rules・ツール設定など |

組織展開では、以下のアプローチが有効です。

**ベースラインテンプレートを配布する方法**

組織標準の `config.yaml` テンプレートを社内 Wiki やファイルサーバに公開し、新規参加者がそこからコピーして利用する運用です。LLM エンドポイントのプレースホルダを入れておくことで、個人が簡単にカスタマイズできます。

```yaml
# config.yaml テンプレート（組織配布用）
# ＜＞ のプレースホルダを自分の環境に合わせて書き換えてください

name: org-continue-config
version: 1.0.0
schema: v1

models:
  - name: 社内 LLM（Chat）
    provider: ollama
    model: <your-model-name>
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - chat
      - edit

  - name: 社内 LLM（Autocomplete）
    provider: ollama
    model: <your-autocomplete-model-name>
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - autocomplete
```

!!! note "テレメトリの設定について"
    <glossary:テレメトリ>は `config.yaml` の `allowAnonymousTelemetry` キーではなく、VS Code の設定（`Ctrl+,` → "Continue: Telemetry Enabled" のチェックを外す）または Continue 拡張の設定画面のトグルで制御します。

**ワークスペース設定をリポジトリ管理する方法**

プロジェクト固有の Rules やツール設定は `.continue/config.yaml` としてリポジトリに含め、Git で管理します。こうすることで、プロジェクトメンバー全員が同じ設定を自動的に利用できます。

!!! note "ユーザー設定とワークスペース設定の優先順位"
    ワークスペース設定はユーザー設定とマージされます。同じキーが両方にある場合、ワークスペース設定が優先されます。

### 更新時の検証フロー

新バージョンの Continue をリリースする際は、以下の 3 ステップで検証することを推奨します。

**ステップ 1: テスト環境での動作確認**

テスト用の開発機に新バージョンをインストールし、以下を確認します。

- <glossary:Chat>・Edit・Autocomplete・<glossary:Agent> の各機能が正常に動作するか
- ローカル LLM エンドポイントへの接続が維持されているか
- テレメトリ無効化設定（`continue.telemetryEnabled: false`）が有効か（[第 4 章](04-telemetry-airgap-verification.md) の検証手順を参照）
- 既存の `config.yaml` が引き続き正しく読み込まれるか

**ステップ 2: 段階的な展開**

問題がなければ、少数の先行ユーザー（5〜10 名程度）に展開し、数日間の試用期間を設けます。フィードバックを収集し、問題がなければ全社展開に進みます。

**ステップ 3: 全社展開とアナウンス**

社内の通知チャネルを通じて、更新内容・インストール手順・問い合わせ窓口を案内します。古いバージョンの VSIX は一定期間保持し、問題発生時にロールバックできるようにしておきます。

---

## ユーザー教育と利用ガイドラインの整備

ツールの設定だけでなく、組織内のユーザーが正しく安全に Continue を使えるよう、教育と仕組みの両面から支援することが重要です。

### 社内向け利用ガイドラインのテンプレート

以下は、社内の Confluence・Wiki・ポータルサイトに掲載する利用ガイドラインのたたき台です。組織の実情に合わせて編集してください。

```markdown
## Continue 利用ガイドライン（社内向け）

### やってよいこと
- 一般的なロジックやアルゴリズムのコード補完・説明
- 単体テストの自動生成
- コードの可読性向上（変数名変更、コメント追加）
- エラーメッセージの解釈と修正案の検討
- 設計パターンの相談

### やってはいけないこと
- 秘密鍵・API キー・パスワードをプロンプトに含める
- 個人情報（氏名・住所・マイナンバー等）を含むコードをそのまま入力する
- NDA を締結しているプロジェクトの機密実装を入力する（法務確認が必要）
- LLM の提案コードを確認せずにそのままマージする

### 不明な点・問題が発生した場合
- 担当窓口: <社内問い合わせ先を記載>
- 報告フォーム: <リンクを記載>
```

### よくある誤用パターンと対策

現場で発生しやすいリスクと、その対策を以下に示します。

**パターン 1: 機密情報を含む環境変数ファイルを `@file` で指定してしまう**

`.env` ファイルに API キーや接続文字列が記載されていると、それが LLM に送信されます。

対策: `.continueignore` に `.env` と `.env.*` を追加する（前述の設定例を参照）。

**パターン 2: LLM の提案コードを検証せずにマージする**

LLM は誤ったコードや、古い API を使ったコードを返すことがあります（<glossary:ハルシネーション>）。

対策: Continue の提案は「叩き台」として扱い、コードレビューのプロセスを省略しないよう、チームのルールに明記する。

**パターン 3: 共有端末で Continue を使い、ログが残る**

共有 PC で使用すると、プロンプトログが他のユーザーに参照される可能性があります。

対策: Continue の使用は個人アカウントが割り当てられた端末に限定し、共有端末での使用を禁止するポリシーを設ける。

**パターン 4: VS Code の自動更新で非承認バージョンがインストールされる**

エアギャップ環境では起きにくいですが、<glossary:プロキシ>経由で Marketplace にアクセスできる場合は注意が必要です。

対策: `settings.json` で VS Code 拡張の自動更新を無効化する。

```json
// VS Code の settings.json
{
  "extensions.autoUpdate": false,
  "extensions.autoCheckUpdates": false
}
```

### フィードバックループの構築

組織全体で Continue を安全かつ効果的に使い続けるには、現場からのフィードバックを運用チームに届ける仕組みが必要です。

**Issue テンプレートの整備**

社内 Git サーバや問題管理ツールに、次のような Issue テンプレートを用意しておくと、報告の質と速度が上がります。

```markdown
## Continue 利用報告

### 報告種別
- [ ] セキュリティ懸念
- [ ] 動作不具合
- [ ] 改善要望
- [ ] その他

### 発生環境
- Continue バージョン: 
- VS Code バージョン: 
- OS: 

### 状況の説明
（何をしていたときに、何が起きたかを記載してください）

### 再現手順
1. 
2. 
3. 

### 関連ファイル・ログ
（ログを添付する場合は、機密情報が含まれていないことを確認の上、添付してください）
```

**定期的な勉強会の実施**

月 1 回程度の社内勉強会を通じて、新機能の紹介・活用事例の共有・よくある質問への回答を行うことで、ユーザーのリテラシーを継続的に高めることができます。勉強会の内容は社内 Wiki に議事録として残し、参加できなかったメンバーも後から参照できるようにしてください。

---

## まとめ

- Continue のデータはローカル LLM エンドポイントにのみ送信される。エアギャップ環境では外部への漏洩リスクは低いが、LLM サーバのログ設定やアクセス権限の管理も併せて確認する
- `.continueignore` と `contextLength` の設定により、機密コードの意図しない送信を防止できる
- Continue のログ（`~/.continue/logs/core.log` および VS Code 出力パネル）は定期的に削除・アーカイブし、機密情報が長期間残存しないよう管理する
- VS Code 拡張と `config.yaml` の配布・更新は、テスト環境での検証 → 段階展開 → 全社展開の 3 ステップで運用する
- 利用ガイドライン・Issue テンプレート・定期勉強会を整備し、技術的な設定とユーザー教育の両輪でセキュリティリスクを低減する

## 次の章へ

次は [第 14 章 トラブルシューティング](14-troubleshooting.md) で、Continue の動作に問題が発生した際のログの確認方法・代表的な症状と切り分け手順・品質チューニングを扱います。

## 参考リンク

- [Continue 公式ドキュメント - Security](https://docs.continue.dev/security)
- [Continue 公式ドキュメント - .continueignore](https://docs.continue.dev/reference/continueignore)
- [MkDocs Material - Admonitions](https://squidfunk.github.io/mkdocs-mater