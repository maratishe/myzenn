---
title: "CopilotやClaude CodeのVSCode拡張での会話コストを計算する"
emoji: "💰"
type: "tech"
topics: ["vscode", "githubcopilot", "claudecode", "llm", "nodejs"]
published: true
---

コードは [github.com/maratishe/vscode-llm-cost-calculator](https://github.com/maratishe/vscode-llm-cost-calculator) にあります。

# CopilotやClaude CodeのVSCode拡張での会話コストを計算する

**所要時間：** 読了 約10分、動作確認 約10分  
**前提条件：** Node.js、GitHub Copilot Chat または Claude Code を入れた VS Code、ディスク上に残っている会話履歴

## 概要

CopilotやClaude CodeをBYOK（自分のAPIキーを使う方式）で利用すると、会話ごとにモデル提供元から課金されます。しかし、エディタ上には「この会話にいくらかかったか」は表示されません。請求は月単位やキー単位の1つの数字として後から届くため、実際の作業と結びつけるのは困難です。

一方で、どちらの拡張機能も、LLMリクエストごとのトークン数を含む詳細なセッションログをローカルディスクに書き出しています。これだけでコストの見積りは可能です。本記事では、そのログを読み取り、SQLiteに保存し、ツール別の日次・月次のコスト、トークン数、リクエスト数を表示する小さなローカルWebアプリを紹介します。

読了後、以下のことができるようになります：

1. Copilot ChatとClaude Codeのセッションログの場所と構造を把握し、パースする
2. キャッシュ読み取り・キャッシュ書き込みを含め、トークン数から正しくコストを計算する
3. 何度実行しても安全な（冪等な）スキャン処理を組み立てる

## 目次

1. **結果：** 実際の画面
2. **データの場所：** ログの保存先と必要なフィールド
3. **コスト計算式：** CopilotとClaude Codeで扱いが異なる理由
4. **保存とスキャン：** 2つのSQLiteファイル、差分スキャン、冪等性
5. **集計とUI**
6. **実行方法と、公開リポジトリから除いたもの**
7. **制約事項**

## 1. 結果

**Scan** を押すと、今月の合計、グラフ（トークン・コスト・リクエスト数を日別または月別）、表が表示されます。こちらがCopilotタブです。

![Copilotタブ：今月17リクエスト、入力800,532トークン、$2.7717](/images/260930-llm-cost-copilot.png)

こちらがClaude Codeタブです。

![Claude Codeタブ：今月365リクエスト、$40.4198](/images/260930-llm-cost-claude-code.png)

（どちらのスクリーンショットも **Pricing** ボタンの追加前のもので、現在のコードにはPricingボタンがあります。）

このスクリーンショットには、設計の大半を説明する重要な点が2つあります。

- Claude Codeタブの2026-08-13の行は、**新規入力が345トークン**しかないのに、**キャッシュが20,075,921トークン**あります。長いエージェント型セッションでは、コンテキストのほぼすべてがプロンプトキャッシュから供給されます。「入力トークン × 入力単価」だけで計算すると、入力側の見積りはほぼ0になり、合計も大きくずれます。
- **In** 列の意味は2つのタブで異なります。Copilotタブの2026-08-29の行は、入力451,009トークンのうち422,478がキャッシュで、**Inにはすでに Cache が含まれて**います。Claude CodeタブのInは新規入力のみです。この違いは次の節で扱います。

## 2. データの場所

| ツール | ログファイル |
|---|---|
| Copilot Chat | `<VS Codeユーザーディレクトリ>/User/workspaceStorage/<workspace>/GitHub.copilot-chat/debug-logs/<session>/main.jsonl` |
| Claude Code | `~/.claude/projects/<project>/<session>.jsonl` |

どちらもJSON Lines形式（1行1イベント）です。ただし、いずれも文書化された安定インターフェースではなく、解析によって得た仕様であり、保守が必要になる前提です。使うフィールドは多くありません。

**Copilot Chat** はモデル呼び出しごとに `llm_request` イベントを書き出します。

```js
if (obj.type === 'llm_request' && obj.attrs) {
    const a = obj.attrs;
    const model   = a.model || 'unknown';
    const inputT  = a.inputTokens  || 0;  // リクエスト全体の合計
    const outputT = a.outputTokens || 0;
    const cachedT = a.cachedTokens || 0;  // inputTの一部
    const freshInputT = Math.max(0, inputT - cachedT);
    // リクエストキー: a.responseId
}
```

**Claude Code** は `assistant` イベントの `message.usage` に数値を持っています。

```js
if (obj.type === 'assistant' && obj.message && obj.message.usage) {
    const u = obj.message.usage;
    const inputT        = u.input_tokens || 0;                  // 新規入力のみ
    const outputT       = u.output_tokens || 0;
    const cachedT       = u.cache_read_input_tokens || 0;
    const cacheCreation = u.cache_creation_input_tokens || 0;
    // リクエストキー: obj.requestId
}
```

各ファイルからは、セッションID、最初と最後のタイムスタンプ、タイトルも取得します。タイトルはCopilotではタイトルファイルがあればそれを、なければ最初のユーザーメッセージ（先頭140文字）を使います。

## 3. コスト計算式

プロバイダーは4種類のトークンを、100万トークンあたりで別々に価格設定しています。

| 種類 | 新規入力に対する典型的な価格 |
|---|---|
| 新規入力 | 1倍 |
| 出力 | 数倍 |
| キャッシュ読み取り | かなり安い（プロバイダーにより0.1〜0.5倍程度） |
| キャッシュ書き込み | 同等か、やや高い |

したがって、1リクエストのコストは次のようになります。

```js
function calcCost(model, t, pricingTable = PRICING) {
    const p = resolvePricing(model, pricingTable);
    return (
        (t.input || 0)         * p.input      / 1e6 +
        (t.output || 0)        * p.output     / 1e6 +
        (t.cached || 0)        * p.cachedRead / 1e6 +
        (t.cacheCreation || 0) * p.cacheWrite / 1e6
    );
}
```

落とし穴は `t.input` に何を渡すかです。Claude Codeは新規入力を直接報告するので、そのまま渡します。Copilotはキャッシュ分を含むリクエスト合計を報告するため、先にキャッシュ分を引きます。引かないと、キャッシュ分が入力単価とキャッシュ読み取り単価で二重に課金されてしまいます。`gpt-5`（入力 / 出力 / キャッシュ読み取りが100万トークンあたり1.25 / 10 / 0.125 USD）での仮の例です。

```
inputTokens 50,000（うちキャッシュ45,000）、outputTokens 1,000
新規入力 = 50,000 - 45,000 = 5,000
コスト   = 5,000 x 1.25/1e6 + 1,000 x 10/1e6 + 45,000 x 0.125/1e6
         = 0.00625 + 0.01 + 0.005625 = $0.021875
```

引き算をしないと、同じリクエストが$0.0781と見積もられ、3倍以上の過大評価になります。

### 価格テーブル

価格は `server.js` 内の1つのオブジェクトにモデル名をキーとして定義しています。

```js
const PRICING = {
    'claude-sonnet-4': { input: 3,    output: 15, cachedRead: 0.3,   cacheWrite: 3.75 },
    'gpt-5':           { input: 1.25, output: 10, cachedRead: 0.125, cacheWrite: 1.25 },
    // ...
    default:           { input: 0,    output: 0,  cachedRead: 0,     cacheWrite: 0 },
};
```

これらは執筆時点のおおよその定価であり、いずれ古くなります。実用上、次の2点を押さえています。

- プロバイダーは `gpt-4o-mini-2024-07-18` のように日付付きのモデルIDを記録することがあります。`resolvePricing` は末尾の `-YYYY-MM-DD` を外して再検索し、それでも見つからなければ `default` を使います。
- **Pricing** ダイアログでは、モデルごとの上書き設定を各DBの `settings` テーブルに保存します。変更後は、保存済みのトークン数から全リクエストの `cost_usd` を再計算するため、ログの再読み込みは不要です。未知のモデルは推測せずコスト0とするため、見落としにくくなります。

## 4. 保存とスキャン

ツールごとに、同一スキーマの別々のSQLiteファイル（`copilot.db`、`claude.db`）を持ちます。

- `sessions`：会話ごとの行（タイトル、最初/最後のタイムスタンプ、集計値）
- `requests`：LLMリクエストごとの行（モデル、4種のトークン数、thinkingトークン、所要時間、`cost_usd`）
- `scanned_files`：各ログファイルの前回スキャン時のパス、mtime、サイズ
- `settings`：価格の上書き設定

2つのツールを別ファイルに分けることで、トークンの意味の違いが混ざらず、どちらか一方だけを削除して作り直すこともできます。

スキャンは、何度実行しても安全になるように設計しています。

1. **未変更ファイルをスキップ。** mtimeとサイズが `scanned_files` と一致するファイルは再読み込みしません。
2. **リクエストキーでUPSERT。** `requests.request_key` は一意（プロバイダーのレスポンスIDまたはリクエストID）で、挿入は `ON CONFLICT(request_key) DO UPDATE` で行います。増えたログの再読み込みや、同じリクエストの重複出現があっても二重計上しません。
3. **セッション合計は再計算。** カウンタを加算するのではなく、ファイルごとに `requests` から再集計します。

結果として「いつスキャンすべきか」に誤りがなくなります。作業のたびに押しても、1週間押さなくても構いません。

## 5. 集計とUI

集計は `requests` に対する単純な `GROUP BY` です。コスト報告における「今日」は自分にとっての今日なので、日付は **ローカル時間** で区切ります。

```sql
SELECT strftime('%Y-%m-%d', ts/1000, 'unixepoch', 'localtime') AS period,
       COUNT(*) AS requests, SUM(input_tokens) AS input_tokens,
       SUM(output_tokens) AS output_tokens, SUM(cached_tokens) AS cached_tokens,
       SUM(cost_usd) AS cost_usd
FROM requests GROUP BY period ORDER BY period;
```

月別は `'%Y-%m'` に変えるだけで、上部のサマリーも同じ式を当月で絞り込んだものです。

フロントエンドは意図的に薄くしてあり、CDN版のAlpine.js、Tailwind、Chart.jsを使った静的HTMLと、少数のJSONエンドポイント（`/api/scan`、`/api/stats`、`/api/summary`、`/api/pricing`）を持つNode.jsバックエンドで構成されています。ページはAPIの返す内容を描画するだけなので、たとえば価格の変更は、POSTを1回送って再取得するだけです。

## 6. 実行方法と、公開リポジトリから除いたもの

```bash
npm install sqlite3
cp webapp/env.example.sh webapp/env.sh    # SECRETを設定（必要ならポートも）
source webapp/env.sh && node webapp/server.js
```

表示されたURL（`http://127.0.0.1:8100/<SECRET>/`）を開き、**Scan** を押します。

このアプリは、もともと個人用の大きなWebアプリの2つのページのうちの1つでした。公開前にLLMコストのページだけを別リポジトリに切り出し、次の点を意図的に変更しています。

- **実際の設定ファイルはコミットしない。** リポジトリにあるのは `env.example.sh` のみで、`env.sh`、`*.db`、`*.pem` は `.gitignore` 対象です。元のアプリの環境ファイルにはAPIキーやパーソナルアクセストークン用の項目がありましたが、このアプリには不要なため、一切コピーしていません。旧TLSの鍵と証明書も持ち込んでいません。
- **ハードコードされたシークレットを廃止。** 元のコードではサーバーとUIの両方に固定のデフォルト値がありました。現在は環境変数から取得し（未設定なら起動ごとにランダム生成）、UIは自身のURLからそれを読み取ります。
- **既定でループバックのみ。** サーバーは `127.0.0.1` にバインドし、ローカル専用ツールには不要なHTTPSサーバーは削除しました。
- **CORSを開放しない。** 元は任意のオリジンを許可していましたが、ページとAPIは同一オリジンなので不要でした。

URL中のシークレットは、ローカルの他のプロセスやページに対する簡易的な防御にすぎず、本格的な認証ではありません。ネットワークには公開しないでください。

もう1点、プライバシーに関する注意です。セッションタイトルは各会話の最初のプロンプトから取得するため、`.db` ファイルにプロンプトの断片が含まれる可能性があります。バージョン管理（現在は除外済み）やスクリーンショットには含めないでください。

## 7. 制約事項

- **請求書ではなく見積りです。** 段階的な価格設定、バッチ割引、長文コンテキストの割増、プロバイダー側の丸めは反映していません。
- **thinkingトークンは保存するが別計上しない。** 出力トークン数にすでに含まれている前提です。
- **ログ形式は非公開仕様。** 拡張機能の更新でフィールド名が変わる可能性があります。数値が急に0になったら、まず `scanCopilotFile` と `scanClaudeFile` を確認してください。
- **価格の陳腐化。** `PRICING` をプロバイダーの価格と照合し、必要ならPricingダイアログで上書きしてください。
- **CopilotはVS Code安定版のみ。** ログの探索は安定版 `Code` のユーザーディレクトリを対象にしています。

## まとめ

要点は小さなものです。拡張機能がすでに書き出しているログを読むこと。トークンの種類を分けて扱い、Copilotの入力トークン数にはキャッシュがすでに含まれることを忘れないこと。リクエスト単位の行を一意キーで保存してスキャンを冪等にし、価格は編集可能なテーブルに置いていつでもコストを再計算できるようにすること。これにより、プロキシや追加の計測を入れなくても、BYOKの利用状況を日別・ツール別・モデル別に見える化できます。

## 参考文献

[1] Anthropic, "Prompt caching." https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching

[2] SQLite, "UPSERT." https://www.sqlite.org/lang_upsert.html

[3] SQLite, "Date And Time Functions." https://www.sqlite.org/lang_datefunc.html

[4] Alpine.js. https://alpinejs.dev/

[5] Chart.js. https://www.chartjs.org/

[6] Tailwind CSS. https://tailwindcss.com/
