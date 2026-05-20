---
title: "第 2 章 前提条件と環境確認"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - prerequisites
  - setup
---

## この章で学ぶこと

- 本チュートリアルを進めるために必要なソフトウェアと最低バージョン要件を把握できる
- ローカル LLM サーバが正常に稼働していることを `curl` コマンドで独立して確認できる
- 「LLM 接続設定が完了している状態」が具体的に何を指すかを理解できる
- 環境確認チェックリストを使って、第 3 章以降に進める準備が整っているかを自己確認できる

---

## 前提となるソフトウェアと必要バージョン

本チュートリアルは、次のソフトウェアが社内環境に展開・配布済みであることを前提としています。
各ツールのインストール方法は本書の対象外です。不明な場合は社内の基盤チームにご確認ください。

| ソフトウェア | 推奨バージョン | 備考 |
| --- | --- | --- |
| VS Code | 1.85 以上 | 下限は Continue 拡張の動作要件に準ずる |
| Continue 拡張 | 0.9 以上 | VSIX 形式で社内配布されていること |
| curl | 7.68 以上 | LLM サーバへの疎通確認に使用 |
| ローカル LLM サーバ | 各製品の最新安定版 | Ollama / vLLM / LM Studio のいずれか |

!!! note "ネットワーク要件"
    本書のすべての手順は、外部インターネットへの接続が一切ない環境で完結します。
    社内 LAN 内でのみ通信が発生します。

### VS Code のバージョン確認

VS Code を起動し、メニューバーの **「ヘルプ」→「バージョン情報」** を選択してください。
表示されたダイアログの「バージョン」欄でバージョン番号を確認できます。

```text
バージョン: 1.89.1
コミット:   <hash>
日付:       2024-05-01T00:00:00.000Z
```

1.85 未満の場合は、社内の VS Code 配布チャネルから最新版を取得してアップデートしてください。

### ローカル LLM サーバの準備状態

本書では、次のいずれかのローカル LLM サーバが社内に配置済みであることを前提とします。

- **Ollama**: REST API をポート `11434` で公開するオープンソースの LLM 実行エンジン
- **vLLM**: OpenAI 互換 API をポート `8000` で公開する高スループット推論エンジン
- **LM Studio**: ローカル GUI ツール。OpenAI 互換サーバをポート `1234` で起動できる

どのサーバを使用しているかは、社内の LLM 基盤チームに確認してください。
以降の手順は、この情報をもとに適切なサンプルを選んでください。

!!! warning "サーバが用意されていない場合"
    ローカル LLM サーバが社内に展開されていない場合、本書の手順は進められません。
    まず社内基盤チームに LLM サーバの展開を依頼してください。

---

## ローカル LLM への疎通確認

Continue をインストールする前に、ローカル LLM サーバが正常に稼働していることを
`curl` コマンドで確認します。
`curl` はコマンドライン（ターミナル）から HTTP リクエストを送るツールです。
Continue を経由せず直接サーバと通信できるため、問題の切り分けに役立ちます。

### curl を使った API 疎通テスト

ターミナル（VS Code 内蔵ターミナルまたは OS 標準のターミナル）を開き、
使用しているサーバに合わせて次のコマンドを実行してください。

**Ollama の場合**

```bash
curl http://<your-internal-llm-host>:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<your-model-name>",
    "messages": [{"role": "user", "content": "こんにちは"}],
    "stream": false
  }'
```

**vLLM の場合**

```bash
curl http://<your-internal-llm-host>:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<your-model-name>",
    "messages": [{"role": "user", "content": "こんにちは"}]
  }'
```

**LM Studio の場合**

```bash
curl http://<your-internal-llm-host>:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<your-model-name>",
    "messages": [{"role": "user", "content": "こんにちは"}]
  }'
```

`<your-internal-llm-host>` と `<your-model-name>` は、社内環境の実際の値に置き換えてください。
LLM サーバが VS Code と同じマシンで動作している場合は `localhost` が使えます。

### レスポンスの確認ポイント

コマンドが成功すると、次のような JSON レスポンスがターミナルに表示されます（Ollama の例）。

```json
{
  "model": "<your-model-name>",
  "message": {
    "role": "assistant",
    "content": "こんにちは！何かお手伝いできることはありますか？"
  },
  "done": true
}
```

**正常と判断できる条件**: `message.content` にモデルの返答テキストが入っていること。

次のエラーが出た場合は、対応する原因を確認してください。

| エラーメッセージの例 | 考えられる原因 |
| --- | --- |
| `Connection refused` | サーバが起動していない、またはポート番号が違う |
| `Could not resolve host` | ホスト名が間違っている、または DNS が解決できない |
| `model not found` | 指定したモデル名がサーバにロードされていない |
| タイムアウト（応答なし） | サーバの GPU / CPU リソースが不足している可能性 |

!!! tip "Windows ターミナルでの注意"
    PowerShell では `curl` は `Invoke-WebRequest` のエイリアスです。
    上記コマンドをそのまま使うには、Git Bash や WSL のターミナルを使用するか、
    PowerShell では `curl.exe` と明示してください。

    ```powershell
    # PowerShell の場合
    curl.exe http://<your-internal-llm-host>:11434/api/chat `
      -H "Content-Type: application/json" `
      -d '{"model":"<your-model-name>","messages":[{"role":"user","content":"こんにちは"}],"stream":false}'
    ```

---

## 「LLM 接続設定が完了している」状態の定義

本書の各章では「LLM 接続設定が完了している前提」で説明を進めます。
具体的には、次の **2 つの条件** を両方満たしている状態を指します。

1. `config.yaml` に LLM エンドポイントの情報が正しく記述されている
2. Continue の Chat パネルからモデルの応答が得られる

それぞれの確認方法を説明します。

### Continue の config.yaml に LLM エンドポイントが設定されている

`config.yaml` は Continue の動作を制御する設定ファイルです（詳細は [第 5 章](05-config-yaml-basics.md) で扱います）。
ここでは最小限の確認ポイントだけを示します。

`config.yaml` のデフォルトの保存場所は次のとおりです。

| OS | パス |
| --- | --- |
| Windows | `%USERPROFILE%\.continue\config.yaml` |
| macOS / Linux | `~/.continue/config.yaml` |

ファイルを開き、`models` セクションに次のような記述があることを確認してください。

```yaml
# Ollama の場合の最小構成例
models:
  - name: "社内 LLM"
    provider: ollama
    model: <your-model-name>
    apiBase: http://<your-internal-llm-host>:11434
```

```yaml
# vLLM / LM Studio（OpenAI 互換）の場合の最小構成例
models:
  - name: "社内 LLM"
    provider: openai
    model: <your-model-name>
    apiBase: http://<your-internal-llm-host>:8000/v1
    apiKey: dummy  # エアギャップ環境では認証不要でも何らかの値が必要な場合がある
```

!!! note "config.yaml が存在しない場合"
    Continue を一度も起動していない場合はファイルが生成されていない可能性があります。
    その場合は先に [第 3 章](03-offline-install.md) で Continue 拡張をインストールしてから
    本節に戻ってきてください。

### Continue から LLM が呼び出せることを確認する

Continue 拡張がインストール済みの場合は、次の手順で Chat 経由の動作確認ができます。

1. VS Code を開き、左サイドバーの Continue アイコン（または `Ctrl+L` / `Cmd+L`）を選択する
2. Chat パネルが開いたら、テキスト入力欄に「こんにちは」と入力して `Enter` を押す
3. モデルからの返答がパネルに表示されれば、接続設定は正常です

!!! warning "Continue がまだインストールされていない場合"
    Continue 拡張をまだインストールしていない場合、この確認は後回しで構いません。
    [第 3 章](03-offline-install.md) のインストール完了後に改めて実施してください。
    一方、前節の `curl` による疎通確認は今すぐ実施できます。

---

## 環境確認チェックリスト

次のチェックリストをすべて確認できたら、第 3 章以降に進む準備が整っています。

- [ ] VS Code のバージョンが 1.85 以上である
- [ ] ローカル LLM サーバの種類（Ollama / vLLM / LM Studio）とホスト名・ポートを把握している
- [ ] `curl` コマンドでローカル LLM サーバにリクエストを送り、モデルの返答が返ってきた
- [ ] `config.yaml` の `models` セクションに LLM エンドポイントが記述されている（Continue インストール済みの場合）
- [ ] Continue の Chat パネルでモデルの返答が表示された（Continue インストール済みの場合）

!!! tip "チェックがつかない項目がある場合"
    `curl` の疎通確認でエラーが出た場合は、社内 LLM 基盤チームにサーバの状態を確認してください。
    `config.yaml` の設定に不明点がある場合は、[第 5 章](05-config-yaml-basics.md) を先に参照するか、
    社内チームが提供している接続手順書を確認してください。

---

## まとめ

- 本チュートリアルに必要なソフトウェアは **VS Code 1.85 以上**、**Continue 拡張**、**ローカル LLM サーバ** の 3 つです
- `curl` コマンドを使えば、Continue を経由せずにローカル LLM サーバの疎通を独立して確認できます
- 「LLM 接続設定が完了している」とは、`config.yaml` にエンドポイントが設定され、Chat パネルで応答が得られる状態を指します
- 本章のチェックリストをすべて満たしてから次章に進むことで、インストール後のトラブルを最小化できます

## 次の章へ

次は [第 3 章 VS Code 拡張のオフラインインストール](03-offline-install.md) で、
インターネット接続なしに Continue 拡張を VSIX ファイルからインストールする手順を扱います。

## 参考リンク

- [Continue 公式ドキュメント — Configuration](https://docs.continue.dev/reference/config)
- [Ollama 公式ドキュメント](https://ollama.com/docs)
- [vLLM 公式ドキュメント](https://docs.vllm.ai/)
- [LM Studio 公式サイト](https://lmstudio.ai/)

!!! note "参考リンクについて"
    上記リンクはすべて外部サイトです。エアギャップ環境からは接続できません。
    事前にオフラインで取得したドキュメントや、社内ミラーを参照してください。
