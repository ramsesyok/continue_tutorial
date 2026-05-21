---
title: "第4章 テレメトリと外部通信の遮断確認"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - telemetry
  - security
  - audit
---

## この章で学ぶこと

- Continue がデフォルトで送信しうる<glossary:テレメトリ>データの種類を理解する
- `config.yaml` と VS Code の設定でテレメトリを無効化する方法を習得する
- VS Code 開発者ツール・<glossary:プロキシ>ログ・<glossary:パケットキャプチャ>の 3 通りの手順で外部通信の有無を検証できるようになる
- 監査担当者に提出できる形でエビデンスを記録・保管する方法を身につける

---

## テレメトリとは

テレメトリ（telemetry）とは、ソフトウェアが動作状況や利用傾向などのデータを、ネットワーク越しに自動送信する仕組みのことです。VS Code 本体や各<glossary:拡張機能>は、開発元が製品改善を目的としてこの仕組みを組み込んでいる場合があります。

<glossary:オンプレミス>・<glossary:エアギャップ>環境では、テレメトリによる外部通信を許可しないことが一般的なセキュリティポリシーとなっています。本章では、Continue におけるテレメトリ設定の無効化と、その効果を検証するための具体的な手順を説明します。

### Continue が収集・送信しうるデータ

Continue はデフォルト設定のままだと、以下のような匿名の使用統計データを送信することがあります。

- 機能の呼び出し回数（<glossary:Chat>、<glossary:Autocomplete>、<glossary:Edit> などの利用頻度）
- エラーが発生した際のスタックトレース（コードの内容は含まない）
- 使用している OS・VS Code バージョン等の環境情報

!!! note "送信されるデータについて"
    Continue の公式ドキュメントでは、送信データにソースコードや個人情報は含まれないと説明されています。ただし、エアギャップ環境ではポリシー上すべての外部通信を遮断することが求められるため、データの内容にかかわらず無効化を推奨します。

### エアギャップ環境における考え方

「テレメトリを無効化すれば外部通信はゼロになる」と断定することは難しい場合があります。設定の変更後も、以下の手順に沿って実際に通信が発生していないことを検証し、その結果を記録することが重要です。本章で説明するアプローチは「無効化設定を施したうえで通信の不在を検証する」という二段構えになっています。

---

## テレメトリを無効化する

### テレメトリの無効化方法

現在の Continue では、テレメトリの有効・無効は **VS Code の設定**で制御します。旧バージョンで使用していた `config.yaml` の `allowAnonymousTelemetry:` キーは現在のバージョンでは使用しません。

VS Code の `settings.json` に以下を追加してください。

```json
// .vscode/settings.json または ユーザー設定
{
  "continue.telemetryEnabled": false
}
```

!!! tip "settings.json の場所"
    VS Code の<glossary:コマンドパレット>（`Ctrl+Shift+P`）から「Preferences: Open User Settings (JSON)」と入力すると、ユーザー設定の `settings.json` を直接開けます。<glossary:ワークスペース>単位で設定する場合は「Preferences: Open Workspace Settings (JSON)」を使用してください。

設定を変更したら VS Code を再起動し、変更を反映させてください。

### VS Code 本体のテレメトリ設定との関係

Continue 拡張のテレメトリに加えて、VS Code 本体のテレメトリも合わせて無効化することを推奨します。

```json
// .vscode/settings.json または ユーザー設定
{
  "continue.telemetryEnabled": false,
  "telemetry.telemetryLevel": "off"
}
```

| 設定箇所 | キー | 影響範囲 |
| --- | --- | --- |
| VS Code `settings.json` | `continue.telemetryEnabled` | Continue 拡張のテレメトリ |
| VS Code `settings.json` | `telemetry.telemetryLevel` | VS Code 本体のテレメトリ |

確実を期すために**両方**を設定することを推奨します。

!!! warning "VS Code 本体のテレメトリについて"
    Continue とは別に、VS Code 本体もテレメトリを送信します。VS Code 本体のテレメトリを無効化するには、`settings.json` に `"telemetry.telemetryLevel": "off"` を追加してください。これは Continue の設定とは独立しているため、忘れずに設定してください。

---

## 外部通信を確認する

ここでは、テレメトリの無効化設定が実際に機能しているかを確認するための 3 つの方法を紹介します。状況に応じて使い分けてください。

### VS Code 開発者ツールを使う方法

VS Code には Chromium ベースの開発者ツールが内蔵されており、拡張機能の通信を直接確認できます。

**手順**

1. VS Code を起動し、Continue が有効な状態にする
2. メニューから [ヘルプ] → [開発者ツールの切り替え]（英語環境では [Help] → [Toggle Developer Tools]）を選択する
3. 開発者ツールが開いたら [Network]（ネットワーク）タブをクリックする
4. フィルター欄に `continue` または `telemetry` と入力する
5. Continue の各機能（Chat の送信、ファイルを開く、補完のトリガーなど）を 1〜2 分間操作する
6. ネットワークタブにリクエストが表示されないこと、または表示されるリクエストがすべてローカル <glossary:LLM> ホスト（`<your-internal-llm-host>`）宛であることを確認する

!!! note "記録のポイント"
    確認が終わったら、ネットワークタブ全体のスクリーンショットを取得してください。「リクエストが 0 件」または「ローカルホスト宛のみ」であることが確認できる画面を保存しておきます。

### プロキシログを使う方法

社内ネットワークにフォワードプロキシが設置されている場合は、そのアクセスログで外部通信の有無を確認できます。

**手順**

1. VS Code を起動し、Continue を操作する（5〜10 分程度）
2. プロキシの管理者コンソール、またはログファイルで当該端末の IP アドレスを発信元としたリクエストを検索する
3. 以下のドメインへのリクエストが記録されていないことを確認する

```text
# Continue のテレメトリ関連として知られる送信先の例
posthog.com
*.posthog.com
continue.dev
*.continue.dev
```

4. 何も記録されていない場合、または LLM ホスト（`<your-internal-llm-host>`）宛のリクエストのみが記録されている場合は、外部通信が発生していないと判断できます。

!!! tip "ログの取得範囲"
    ログは Continue を操作している間の時刻帯（例: 14:00〜14:10）を絞り込んで取得すると、ノイズが少なくなります。

### パケットキャプチャを使う方法

より詳細な確認が必要な場合は、ネットワークパケットキャプチャツールを使います。エアギャップ環境にパケットキャプチャツールをあらかじめオフライン配布しておく必要があります。

#### Wireshark を使う場合

Wireshark（バージョン 4.x 推奨）を使ったキャプチャ手順を示します。

**手順**

1. Wireshark を起動し、VS Code が使用しているネットワークインターフェース（例: `eth0`、`en0`）を選択する
2. キャプチャフィルターに以下を入力してキャプチャを開始する

```text
not host <your-internal-llm-host>
```

これにより、LLM ホスト宛の通信を除外したトラフィックだけが表示されます。

3. VS Code 上で Continue を 5〜10 分間操作する
4. キャプチャを停止し、パケット一覧を確認する
5. 外部 IP アドレス宛のパケットが存在しないことを確認する

#### `tcpdump` を使う場合

Linux 環境では `tcpdump` でも同様の確認が可能です。

```bash
# LLM ホスト以外への TCP 通信を記録する（実行には root 権限が必要）
sudo tcpdump -i eth0 -w /tmp/continue-check.pcap \
  not host <your-internal-llm-host>
```

キャプチャファイル（`.pcap`）は Wireshark で開いて内容を確認できます。

!!! warning "キャプチャデータの取り扱い"
    パケットキャプチャにはローカル LLM とのやり取り（コードの断片を含む可能性あり）が記録されます。取得したファイルは目的の確認が終わり次第、安全に削除してください。

---

## 監査エビデンスの記録

### 確認結果のスクリーンショット保存

各検証手順で取得したスクリーンショットは、以下の命名規則で保存してください。

```text
形式: YYYYMMDD_<手法>_<内容>.png

例:
  20260520_devtools_network-tab-empty.png    # 開発者ツール・ネットワークタブが空
  20260520_proxy-log_no-external-request.png # プロキシログに外部リクエストなし
  20260520_wireshark_no-external-packet.png  # Wireshark で外部パケットなし
```

保管先は社内のセキュリティ管理システム、または監査用ドキュメントフォルダに格納してください。

### エビデンスチェックリスト

監査担当者への提出前に、以下のチェックリストをすべて満たしていることを確認してください。

```text
【Continue テレメトリ無効化 確認チェックリスト】

実施日: ____年____月____日
確認者: ________________
対象端末: ________________
VS Code バージョン: ________________
Continue バージョン: ________________

■ 設定変更
  [ ] settings.json に continue.telemetryEnabled: false を記載した
  [ ] settings.json に telemetry.telemetryLevel: "off" を記載した（VS Code 本体分）
  [ ] VS Code を再起動して設定を反映した

■ 確認（以下のいずれか 1 つ以上を実施）
  [ ] VS Code 開発者ツールのネットワークタブで外部通信がないことを確認した
  [ ] プロキシログで対象端末からの外部通信がないことを確認した
  [ ] パケットキャプチャで外部宛パケットがないことを確認した

■ エビデンス保存
  [ ] スクリーンショットまたはログを指定フォルダに保存した
  [ ] ファイル名に日付・手法・内容を含めた
```

---

## まとめ

- Continue はデフォルトで使用統計などのテレメトリを送信することがあるが、コードや個人情報は含まれない
- VS Code の `settings.json` に `continue.telemetryEnabled: false` と `telemetry.telemetryLevel: "off"` を設定することで無効化できる
- 無効化後は、開発者ツール・プロキシログ・パケットキャプチャのいずれかの手順で外部通信がないことを検証する
- 検証結果はスクリーンショットやログとして保存し、エビデンスチェックリストとあわせて監査担当者に提出する

## 次の章へ

次は [第5章 config.yaml の基本構造](05-config-yaml-basics.md) で、Continue 全体の設定ファイルの書き方を詳しく学びます。モデルの接続先や各機能のパラメータをどのように記述するかを理解できます。

## 参考リンク

- [Continue 公式ドキュメント — Telemetry](https://docs.continue.dev/telemetry)
- [VS Code テレメトリ設定](https://code.visualstudio.com/docs/getstarted/telemetry)
- [Wireshark 公式サイト](https://www.wireshark.org/)
- [Material for MkDocs — Admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/)
                                                                                                         