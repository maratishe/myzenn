---
title: "VS Code CopilotのAnthropic BYOKでコンテキストウィンドウを200kに固定する"
emoji: "🪟"
type: "tech"
topics: ["vscode", "githubcopilot", "anthropic", "claude", "llm"]
published: true
published_at: 2026-10-12 00:00
---

# VS Code CopilotのAnthropic BYOKでコンテキストウィンドウを200kに固定する

**公開日：** 2026年10月12日  
**所要時間：** 読了 約5分、設定 約5分  
**前提条件：** VS Code（動作確認は 1.141.0）、GitHub Copilot Chat、AnthropicのAPIキー

## 概要

私の経験では、**Claude Sonnet 5.5 を High（思考レベル）・200kコンテキスト**で使う構成が、最もコストパフォーマンスが良いと感じています。ジョブは複数のリクエストに分割されますが、そのぶんウィンドウは「必要なものだけ」で埋まり、効率が良いからです。GitHubアカウントの請求を継続的に見てきた上での、あくまで個人的な実感です。

ところが、思考レベルとコンテキスト上限を設定できるのは、VS Code Copilotの**ネイティブモード**（自分のGitHubログインで使うモード）だけです。

- VS Code CopilotのBYOKモード：設定できない
- VS Code上のClaude Code拡張機能：設定できない
- Claude Code Desktop：設定できない

特にBYOKでは、各モデルが提供可能な最大のウィンドウで公開されます。多くのモデルは**1M**です。本記事では、VS Codeの **Custom Endpoint** 機能を使って、AnthropicのAPIキーのまま **200kウィンドウ・High** のモデルを登録する方法を紹介します。

## 1. 何が問題か

1Mウィンドウでは、会話が進むにつれてウィンドウが埋まっていきます。ソースコードやツール出力が大量に溜まり、本来は不要なものにまで入力トークンの料金を払うことになります。

私の場合、BYOKの請求額は、**同じSonnet 5.5**をネイティブのCopilotで使った場合の**5〜10倍**でした。これは厳密に計測した実験ではなく、自分の請求を見た実感です。ただし、200kに切り替えてからは、200kに収まらないプロジェクトや会話にまだ出会っていません。最近のプロンプトは、作業を複数リクエストに賢く分割してくれます。

なお、200kにしても「セッション全体の支出の上限」になるわけではありません。上限がかかるのは**1リクエストあたりのコンテキスト**です。

## 2. 方針：Custom Endpoint を使う

標準のAnthropic BYOKの定義に `contextSize: 200000` や `contextWindow: 200000` を足してみましたが、UIの1M表示は変わりませんでした。そこで、別経路の **Custom Endpoint** を使います。着想は [VS Code issue #329892: Expose contextSizeOptions for BYOK models](https://github.com/microsoft/vscode/issues/329892) です。

## 3. 手順

1. コマンドパレットで **`Chat: Manage Language Models`** を実行します。
2. **Add Models → Custom Endpoint** を選びます。

![Add Models メニューの Custom Endpoint](/images/261012-byok-custom-endpoint.png)

3. プロバイダーに分かりやすい名前を付けます（例：`claude-byok-sonnet-5.5`）。
4. AnthropicのAPIキーをVS CodeのUIから入力します。キーはシークレットとして保存され、`${input:chat.lm.secret.…}` という参照がJSONに書き込まれます。**自分の環境で生成された参照をそのまま残してください。**生のAPIキーをJSONに貼ってはいけません。
5. API typeは **Messages** を選びます。
6. 生成された `chatLanguageModels.json` のひな形を、次の節のJSONで置き換えます。
7. 保存し、必要なら **`Developer: Reload Window`** を実行します。モデルピッカーで新しいモデルを選びます。

## 4. 設定JSON

Custom Endpointを作成した直後のひな形は、次のように最小限です。

```json
[
  {
    "name": "claude-byok-sonnet-5.5",
    "vendor": "customendpoint",
    "apiKey": "${input:chat.lm.secret.xxxxxxxx}",
    "apiType": "messages",
    "models": [
      {
        "id": "",
        "name": "Sonnet 5.5 (High+200k)",
        "url": "",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 128000,
        "maxOutputTokens": 16000
      }
    ]
  }
]
```

これを、次の内容に置き換えます。

```json
[
  {
    "name": "claude-byok-sonnet-5.5",
    "vendor": "customendpoint",
    "apiKey": "${input:chat.lm.secret.xxxxxxxx}",
    "apiType": "messages",
    "models": [
      {
        "id": "claude-sonnet-5-5",
        "name": "Sonnet 5.5 (High+200k)",
        "url": "https://api.anthropic.com/v1/messages",
        "toolCalling": true,
        "vision": true,
        "thinking": true,
        "supportsReasoningEffort": ["high"],
        "reasoningEffortFormat": "messages",
        "contextWindow": 200000,
        "maxOutputTokens": 64000
      }
    ]
  }
]
```

注意点：

- `apiKey` の `xxxxxxxx` の部分は**ダミー**です。自分のVS Codeが生成した参照に置き換えてください（上記の手順4）。
- `id` には**Anthropic APIのモデルID**（`claude-sonnet-5-5`、ハイフン区切り）を指定します。ネイティブのCopilotが使うドット区切りの名前（`claude-sonnet-5.5`）とは表記が異なるので注意してください。自分の組織で利用できるIDが異なる場合は、そちらを指定します。
- ひな形にあった `maxInputTokens` は削除し、代わりに `contextWindow` で200000を指定しています。

### `maxOutputTokens` が必要な理由

最初は出力の制限を避けるため、`maxInputTokens` と `maxOutputTokens` の両方を省略しました。モデルは選択でき、UIにも200Kと表示されましたが、実際のリクエストはリトライの末に次のエラーで失敗しました。

```text
400 invalid_request_error: max_tokens: Input should be a valid integer
```

AnthropicのMessages APIは、**リクエストごとに**数値の `max_tokens` を要求します。`"maxOutputTokens": 64000` を指定すると解消しました。

この64kは**1回のAPI応答の上限**であり、プロンプトやエージェントのセッション全体の上限ではありません。1つのプロンプトが、ファイル読み込みやツール実行を挟んで複数のモデル呼び出しになることがあり、上限は呼び出しごとに適用されます。数行の編集に64k使うわけでもありません。実際に消費されるのは、モデルが生成した分です。

この設定でのVS Codeの予算の扱いは、次のとおりです。

| 項目 | トークン数 |
|---|---|
| コンテキストウィンドウ全体 | 200,000 |
| 応答用の最大確保量 | 64,000（APIリクエストごと） |
| 導出される入力の配分 | 136,000 |
| セッション全体の生成量 | 64,000で頭打ちにはならない |

つまり、**200kのコンテキスト ≠ 200kの入力**です。ネイティブCopilotの200Kプリセットが同じ量の出力を確保しているかは未確認で、このカスタム設定のほうが入力側に保守的な可能性があります。

## 5. 1つのエンドポイントには1つのモデルだけ

JSONは、プロバイダー（エンドポイント）とモデルの**2階層**構造です。

```text
chatLanguageModels.json
└── Custom Endpoint（プロバイダー）
    ├── name, vendor, apiKey, apiType
    └── models: [...]
        └── モデル定義
            ├── id, name, url
            ├── toolCalling, vision
            ├── thinking / reasoning-effort
            └── contextWindow, maxOutputTokens
```

`models` は配列なので複数書けそうですが、**同じモデルIDでMediumとHighの2エントリ**を入れたところ、最初のMediumしか反映されませんでした。Highの1エントリだけにすると、Highが表示されました。原因がモデルIDの重複なのかは切り分けていません。確実に動く構成としては、**1つのCustom Endpoint = 1プロバイダー + 1モデル**として扱い、別バリアントが必要なら別のプロバイダーを作って個別に試すことをおすすめします。

## 6. 確認結果

![Session Info の Context Window が 200K 表示](/images/261012-byok-context-window.png)

- モデルピッカーにカスタムプロバイダーが表示され、詳細は **Max context: 200K** でした。
- Session Info の Context Window は、200Kに対する使用量（例：**57.5K / 200K**）を表示しました。
- 設定をHighの1エントリにすると、チャットのフッターに **High** と表示され、その後のプロンプトも正常に動作しました。
- あるセッションでは、使用量57.5Kのうち、ツール定義だけで約**19.3%**を占めていました。リポジトリ以外のコンテキストのオーバーヘッドの一例です。

## 7. 未確認のこと

- ネイティブCopilotのSonnet 5.5 High (200K) と、**送信される思考パラメータが完全に同一か**は確認していません。UIにHighと表示され、リクエストが成功することまでを確認しています。
- ネイティブの `max_output_tokens`、コンパクション（圧縮）の挙動、キャッシュ、タスクごとのコストの完全な一致も未確認です。
- 200kは**リクエストごと**のコンテキスト上限です。長いエージェント実行では、重なり合うコンテキストが何度も送信されるため、キャッシュ、コンパクション、ツールの挙動は引き続き影響します。比較するときは、同じタスクで入力・キャッシュ読み取り・キャッシュ書き込み・出力のトークン数を見比べてください。

## まとめ

VS Code CopilotのBYOKで200kのウィンドウを使いたいときは、**1つのエンドポイントに1つのモデル**を定義するCustom Endpointを使い、`contextWindow` に `200000`、`maxOutputTokens` に有効な数値（検証値は `64000`）を指定します。プロキシサーバーも環境変数も使わず、自分のAnthropic APIキーのまま、VS Code Copilot Agentで200kの上限とHighの表示が得られます。
