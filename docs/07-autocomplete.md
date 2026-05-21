---
title: "第7章 Autocomplete（タブ補完）を使いこなす"
type: chapter
tags:
  - continue
  - vscode
  - on-premise
  - air-gap
  - autocomplete
  - performance
---

## この章で学ぶこと

- Autocomplete（タブ補完）の動作原理と、Chat 機能との根本的な違いを理解する
- `config.yaml` に補完専用モデルを設定し、ローカル LLM と接続する
- 実際に VS Code でタブ補完を操作するハンズオンを体験する
- レイテンシと精度をチューニングして、エアギャップ環境に最適化する
- 補完が効きやすいコードの書き方と、苦手なケースへの対処法を身につける

---

## Autocomplete とは何か

Autocomplete（オートコンプリート）とは、コードを入力している途中でローカル LLM が次に来るべきコードを予測し、候補として表示する機能です。VS Code の画面上では薄いグレーのテキストとして候補が表示され、Tab キーを押すだけで確定できます。

従来の IDE が持つ補完機能（メソッド名やスニペットの候補表示）とは異なり、Continue の Autocomplete は **文脈を読んだ自由形式のコード生成** を行います。関数の引数リストを補完したり、コメントの内容から実装全体を生成したりと、より高度な補完が可能です。

### タブ補完の仕組み

Continue の Autocomplete は、**FIM（Fill-In-the-Middle）** と呼ばれる推論方式を使っています。FIM とは「カーソルより前のコード（Prefix）」と「カーソルより後のコード（Suffix）」の両方をモデルに渡し、その間に入るべきコードを生成させる手法です。

```
[Prefix: カーソルより前のコード]
  ↓
[モデルが中間部分を生成]
  ↓
[Suffix: カーソルより後のコード]
```

この仕組みにより、関数の途中や引数リストの途中など、「前後の文脈がある場所」でも自然な補完が行われます。FIM をサポートしているモデルは補完専用モデルとして適切に機能しますが、すべてのモデルが FIM に対応しているわけではありません。モデルを選ぶ際は FIM 対応の有無を確認してください。

処理の流れは次のとおりです。

1. キーストロークのたびに `debounceDelay`（後述）で設定した時間だけ待機する
2. 待機後、カーソル前後のコードをローカル LLM に送信する
3. モデルが補完候補を生成して VS Code に返す
4. エディタ上に薄いグレーのテキストとして表示される
5. Tab キーで確定、`Esc` キーで却下する

### Chat 機能との違い

Autocomplete と Chat は、どちらも LLM を使ってコードを生成しますが、**目的・モデルの種類・要求されるレスポンス速度** がまったく異なります。

| 観点 | Autocomplete | Chat |
| --- | --- | --- |
| 用途 | 入力中のコードをリアルタイムで補完 | 質問への回答、説明、設計の相談 |
| 応答速度の要件 | **非常に高い**（1〜2 秒以内が目安） | 比較的緩やか（5〜10 秒でも許容される） |
| 使用するモデル | 補完専用の小型・高速モデル | 高精度な汎用 LLM |
| 出力の性質 | 短い・ピンポイント・FIM 対応必須 | 長い説明・マルチターン対話 |
| コンテキスト量 | 少量（カーソル前後のみ） | 大量（ファイル全体・複数ファイル） |

この違いから、**Autocomplete と Chat には別々のモデルを割り当てることが推奨されます**。Chat には精度重視の大型モデルを、Autocomplete にはレスポンス速度重視の小型モデルを使うことで、それぞれの用途に最適なパフォーマンスが得られます。同一モデルを両方に使い回すと、補完のたびに大きなモデルへリクエストが飛び、開発体験が著しく低下します。

!!! tip "モデルを分ける理由"
    「補完するだけなのに別モデルが必要？」と思うかもしれません。しかし、コードを書いている最中に 5 秒間処理が止まると、思考の流れが途切れて生産性が下がります。Autocomplete は「使っていることを意識させない」くらい速く動いてこそ価値があります。

---

## config.yaml で補完を設定する

### 補完専用モデルを選ぶ

Autocomplete に適したモデルは、次の条件を満たすものです。

- **FIM（Fill-In-the-Middle）に対応している**
- **パラメータ数が小さく、推論が速い**（目安: 1B〜7B 程度）
- **コード生成に特化した事前学習を受けている**

代表的な補完専用モデルとして、以下のようなものがあります。

| モデル名 | 特徴 |
| --- | --- |
| `starcoder2:3b` | StarCoder 2 の 3B 版。軽量で速い |
| `deepseek-coder:1.3b` | DeepSeek-Coder の最小版。非常に高速 |
| `deepseek-coder:6.7b` | 精度と速度のバランスが取りやすい |
| `codellama:7b-code` | Code Llama の FIM 対応版 |

エアギャップ環境では、これらのモデルを事前に社内サーバへオフラインで配布しておく必要があります。モデルファイルの入手・検証・配布の手順は、社内のモデル配布運用ルールに従ってください。

!!! warning "チャットモデルは補完に使わない"
    `llama3`、`mistral`、`qwen` などの汎用チャットモデルは FIM に対応していないものが多く、補完の品質が大きく低下します。補完には必ず補完専用モデルを使用してください。

### エンドポイントの記述例

`config.yaml` の `models` リストに `roles: [autocomplete]` を指定したエントリを追加することで、補完専用モデルを設定できます。以下に Ollama / vLLM / LM Studio それぞれの最小構成例を示します。

**Ollama を使う場合**

```yaml
models:
  - name: Autocomplete（Ollama）
    provider: ollama
    model: starcoder2:3b
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - autocomplete
```

**vLLM を使う場合**

```yaml
models:
  - name: Autocomplete（vLLM）
    provider: openai
    model: <your-model-name>
    apiBase: http://<your-internal-llm-host>:8000/v1
    apiKey: dummy
    roles:
      - autocomplete
```

!!! note "vLLM の apiKey について"
    vLLM はデフォルトで認証なしで動作しますが、Continue の `openai` プロバイダーは `apiKey` フィールドが必須のため、ダミー文字列を設定します。実際の認証設定は vLLM 側の設定に従ってください。

**LM Studio を使う場合**

```yaml
models:
  - name: Autocomplete（LM Studio）
    provider: lmstudio
    model: <your-model-name>
    apiBase: http://<your-internal-llm-host>:1234
    roles:
      - autocomplete
```

Chat モデルと組み合わせた全体の `config.yaml` 構成イメージは次のとおりです。

```yaml
models:
  - name: Chat モデル
    provider: ollama
    model: <your-chat-model-name>
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - chat
      - edit

  - name: 補完モデル
    provider: ollama
    model: starcoder2:3b
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - autocomplete
```

設定を保存すると、Continue が自動的に新しいモデル設定を読み込みます。VS Code の再起動は不要です。

---

## ハンズオン：タブ補完を体験する

ここでは、実際に VS Code でタブ補完を操作してみます。`config.yaml` の設定が完了し、補完モデルが社内サーバで動作していることを前提とします。

### 補完の呼び出しと確定

まず、新しい Python ファイル（例: `hello.py`）を VS Code で開き、次のように入力してみてください。

```python
def greet(name: str) -> str:
    """指定した名前に挨拶するメッセージを返す。"""

```

関数のドキュメント文字列（`"""`）の後でカーソルを止めると、しばらくして薄いグレーの文字で補完候補が表示されます。

- **Tab キー**: 候補を確定してコードに挿入します
- **`Esc` キー**: 候補を却下して何もしません
- **`Alt + \`**（Windows/Linux）または **`Option + \`**（macOS）: 候補が表示されていないときに手動で補完をトリガーします

!!! tip "補完が表示されない場合"
    補完候補が現れない場合は、VS Code の右下ステータスバーにある Continue のアイコンを確認してください。スピナーが回り続けている場合はモデルへのリクエストが処理中です。エラーが表示されている場合は [第 14 章](14-troubleshooting.md) を参照してください。

### マルチライン補完を使う

Autocomplete はひと続きの長いコードブロックも生成できます。次のような Python コードを書いてみてください。

```python
# リストの各要素を2乗して合計を返す関数
def sum_of_squares(numbers):
```

関数のシグネチャ行の末尾で止めると、モデルが関数本体全体を補完候補として提示することがあります。Tab キーで確定し、意図どおりのコードになっているか確認してください。

```python
# リストの各要素を2乗して合計を返す関数
def sum_of_squares(numbers):
    return sum(x ** 2 for x in numbers)  # ← 補完で生成された例
```

!!! warning "補完結果は必ず確認する"
    補完で生成されたコードが常に正しいとは限りません。特にエッジケースの処理や型の扱いについては、目視でレビューしてからコードベースに残すようにしてください。LLM の補完はあくまでも「提案」です。

---

## レイテンシと精度のチューニング

エアギャップ環境では、クラウドサービスと異なり GPU リソースが限られている場合がほとんどです。ここでは、`config.yaml` と LLM サーバの設定で調整できるパラメータを解説します。

### レスポンス速度に影響する設定

**`debounceDelay`**（ミリ秒）

キー入力後、何ミリ秒待ってから補完リクエストを送るかを制御します。デフォルトは `300`（0.3 秒）です。値を大きくするとリクエスト頻度が下がり、サーバ負荷が減ります。

`debounceDelay` などのオプションは、`models` リストの該当エントリ内の `autocompleteOptions:` フィールドに記述します。

```yaml
models:
  - name: 補完モデル
    provider: ollama
    model: starcoder2:3b
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - autocomplete
    autocompleteOptions:
      debounceDelay: 500   # 0.5 秒待ってからリクエスト
```

**`maxPromptTokens`**

LLM に送るプロンプト（Prefix + Suffix）の最大トークン数です。デフォルトは `1024` 程度です。値を下げると入力量が減り、推論が速くなりますが、補完の文脈が短くなって精度が落ちる場合があります。

```yaml
    autocompleteOptions:
      maxPromptTokens: 512
```

### 精度に影響する設定

**`temperature`**

出力のランダム性を制御します。`0` に近いほど決定論的（同じ入力には同じ出力）になり、補完の一貫性が上がります。補完用途では `0` 〜 `0.2` 程度が推奨されます。

```yaml
    autocompleteOptions:
      temperature: 0.0
```

**`maxSuffixPercentage`**

プロンプト全体のうち、Suffix（カーソル後のコード）に割り当てる割合の上限です。デフォルトは `0.25`（25%）です。ファイル末尾付近で補完する場合は Suffix が短くなるため、この値の影響は限定的です。

```yaml
    autocompleteOptions:
      maxSuffixPercentage: 0.25
```

### エアギャップ環境での最適化指針

社内 GPU サーバのリソースが限られている場合は、以下の方針でチューニングすることをおすすめします。

1. **補完モデルと Chat モデルを別プロセス（または別サーバ）で起動する**: 1 台のサーバで両方を動かすと、Chat のリクエスト処理中に補完が詰まります
2. **補完モデルは 3B 以下を第一候補にする**: 7B モデルでも高速な GPU なら問題ありませんが、量子化（GGUF Q4 等）も活用して推論速度を稼ぐ
3. **`debounceDelay` を 400〜600ms に設定する**: 入力の速いエンジニアでも入力が一段落してからリクエストが飛ぶため、無駄なリクエストを削減できる
4. **`maxPromptTokens` を 512 程度に抑える**: 補完に必要な文脈は通常それほど長くないため、過大に設定するとかえって遅くなる

以下は、上記の指針をまとめた設定例です。

```yaml
models:
  - name: 補完モデル
    provider: ollama
    model: starcoder2:3b
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - autocomplete
    autocompleteOptions:
      debounceDelay: 500
      maxPromptTokens: 512
      temperature: 0.0
```

---

## 補完を効果的に使うコツ

### 補完が効きやすいコードの書き方

LLM は「文脈の手がかり」が多いほど精度の高い補完を生成します。次の習慣をつけると補完の品質が上がります。

**コメントを先に書く**

実装の意図をコメントで記述してからカーソルを止めると、モデルがコメントを読んで実装を生成します。

```python
# 入力リストから重複を除き、昇順にソートして返す
def unique_sorted(items):
```

**型注釈を付ける**

引数や戻り値の型を明示することで、モデルが型に合った実装を生成しやすくなります。

```python
def find_user(user_id: int) -> dict[str, str] | None:
```

**意味のある変数名・関数名を使う**

`x`、`tmp`、`data` のような汎用的な名前より、`invoice_total`、`fetch_active_users` のような具体的な名前の方が、モデルが文脈を読みやすくなります。

### 補完が苦手なケースと対処法

Autocomplete が精度良く機能しない場面もあります。以下のケースでは Chat 機能（[第 6 章](06-chat-basics.md)参照）への切り替えを検討してください。

**独自のドメイン知識が必要なコード**

社内固有の業務ロジックや、社内ライブラリ固有の API を多用するコードは、モデルの学習データに含まれていないため、補完の精度が低くなります。Chat で「このライブラリの API を説明して」と聞きながら実装する方が効率的です。

**非常に長いファイルの中間部分**

ファイルが数千行以上になると、カーソル前後の文脈だけでは補完の精度が落ちます。リファクタリングによるファイル分割か、Chat の `@file` コンテキスト（[第 6 章](06-chat-basics.md)参照）を活用してください。

**自動生成されたコードや設定ファイル**

Protobuf から生成された Java コードや、ツールが出力した JSON スキーマのような自動生成ファイルは、補完に向いていません。こうしたファイルは補完を無効化しておくと、不要なリクエストを削減できます。

`config.yaml` で特定のファイルパターンを補完対象から除外できます。

```yaml
models:
  - name: 補完モデル
    provider: ollama
    model: starcoder2:3b
    apiBase: http://<your-internal-llm-host>:11434
    roles:
      - autocomplete
    autocompleteOptions:
      disableInFiles:
        - "**/*.generated.*"
        - "**/proto/**"
        - "**/__generated__/**"
```

!!! note "補完を一時的に止めたいとき"
    VS Code のステータスバーにある Continue のアイコンをクリックすると、補完のオン／オフを素早く切り替えられます。長いドキュメントを書くときなど、補完が邪魔に感じる場面で活用してください。

---

## まとめ

- Autocomplete は FIM 方式でカーソル前後のコードを推論し、Tab キーで確定するリアルタイム補完機能です
- Chat とは役割・モデル・求められる速度がまったく異なるため、**補完には `roles: [autocomplete]` で専用の小型モデルを設定する**ことが重要です
- `config.yaml` の `models` リストに `roles: [autocomplete]` のエントリを追加し、`autocompleteOptions:` でレイテンシ・精度の細かなチューニングができます
- コメント先行・型注釈・具体的な命名という習慣が、補完の精度を高める最大の近道です
- 独自ドメインの複雑なコードや自動生成ファイルは補完の対象外とし、Chat と使い分けることで生産性が向上します

## 次の章へ

次は [第 8 章 Edit 機能でコードを書き換える](08-edit.md) で、既存のコードをインラインで書き換える Edit 機能を扱います。補完が「書きながら生成する」機能であるのに対し、Edit は「書いた後で修正・改善する」機能です。リファクタリングや不具合修正のワークフローを体験しましょう。

## 参考リンク

- [Continue 公式ドキュメント — Tab Autocomplete](https://docs.continue.dev/features/autocomplete)
- [Continue 公式ドキュメント — Config Reference（tabAutocompleteModel）](https://docs.continue.dev/reference/config)
- [StarCoder 2 モデルカード（Hugging Face）](https://huggingface.co/bigcode/starcoder2-3b)
- [DeepSeek-Coder モデルカード（Hugging Face）](https://huggingface.co/deepseek-ai/deepseek-coder-1.3b-instruct)
