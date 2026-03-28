---
title: "フラットDBにおけるグラフ構造を用いたファイルライフサイクル追跡"
emoji: ""
type: "tech"
topics: ["multidimensional", "graph", "tracing"]
published: true
---


# ハンズオン：フラットDBにおけるグラフ構造を用いたファイルライフサイクル追跡

**所要時間：** 読了 約20〜30分、ハンズオン実施 約1〜2時間  
**前提条件：** Node.js (v16以上)、SQLite3 CLI、テキストエディタまたはVS Code  
**データセット：** 匿名化済みの業務ログ（JSON形式で提供）

## 概要

多くのエンタープライズシステムでは、1つのファイル（例：会議録音の音声ファイル）が複数の処理ステージを通過し、異なる従業員によって操作され、複数のセッションに跨って参照されます。このようなファイルのライフサイクル全体を把握するには、通常グラフベースの分析が必要ですが、Neo4j [1] や TigerGraph [2] のような専用グラフデータベースの導入は、既存インフラとの整合性やコスト面で必ずしも現実的ではありません。

本ハンズオンでは、その軽量な代替手段を紹介します。**フラットなリレーショナルデータベース（SQLite [3]）の1つのテキストフィールドに多次元のグラフ関係を埋め込み**、SQL、Mermaid [4]、Graphviz [5] を使ってそれらの関係を抽出・可視化する手法です。

本セッション終了後、以下のことができるようになります：

1. フラットDBスキーマにグラフ的な関係性をエンコードする方法の理解
2. 再帰SQL（Recursive CTE）を用いて複数値フィールドの分割・重複排除・追跡を行うクエリの作成
3. メトロマップ形式のダイアグラムによるトレース関係の可視化


## 目次

1. **データの確認：** 複数の次元（ファイル名、従業員、セッション、処理ステージ等）を持つ実際のログエントリ
2. **課題：** なぜ複数の次元を1つのフィールドに埋め込む必要があるか
3. **解法1：** グラフノードの定義（ファイル、セッション、処理ステージ、従業員）
4. **解法2：** 各ユニーク識別子をMD5ハッシュで特定し、DB内の1フィールドに集約
5. **解法3：** 再帰SQLによる分割・重複排除・追跡クエリ
6. **解法4：** ユニーク識別子の一括抽出とメトロマップによる可視化
7. **参考文献**



## (1) データの確認：業務ログにおける複数の次元

本システムの各ログエントリは、**複数の次元を同時に参照する**イベントを記録しています：

| 次元 | フィールド | 例 |
|------|-----------|-----|
| **ファイル** | `file_id`, `file_ids`, `deleted_files` | `file_0008.mp3` |
| **従業員** | `employee_code` | `POOW` |
| **セッション** | `session_id`, `trace_ids` | `7de98ea6e91a` |
| **処理ステージ** | `span_id` | `BE.function2.get` |
| **場所** | `location` | `location1-backend-somewhere` |
| **時刻** | `created_at` | `2026-02-04T08:48:56.816835+00:00` |

以下は匿名化済みデータセットからの3つの代表的なログエントリです。1つの `BE.function2.get` 呼び出しが9つのファイルを参照し、`BE.file.new` エントリがファイルを `trace_ids` を介してセッションにリンクしている点に注目してください：

```
...
{
    "id": 398279,
    "level": "INFO",
    "created_at": "2026-02-04T08:48:56.816835+00:00",
    "employee_code": "POOW",
    "trace_ids": "",
    "location": "location1-backend-somewhere",
    "span_id": "BE.function2.get",
    "details": "{\"date\":null,\"cutoff_date\":\"2026-01-29 00:00:00\",\"file_ids\":[\"file_0001.mp3\",\"file_0002.mp3\",\"file_0003.mp3\",\"file_0004.mp3\",\"file_0005.mp3\",\"file_0006.mp3\",\"file_0007.mp3\",\"file_0008.mp3\",\"file_0009.mp3\"]}",
    "_bulk_data": null,
    "_bulk_data_error": null
},
{
    "id": 398277,
    "level": "INFO",
    "created_at": "2026-02-04T08:48:48.618631+00:00",
    "employee_code": "POOW",
    "trace_ids": "7de98ea6e91a",
    "location": "location1-backend-somewhere",
    "span_id": "BE.file.new",
    "details": "{\"file_id\":\"file_0008.mp3\",\"date\":\"20260204\",\"meeting\":\"45872081\",\"some_flag\":false}",
    "_bulk_data": null,
    "_bulk_data_error": null
},
{
    "id": 395996,
    "level": "INFO",
    "created_at": "2026-01-31T00:01:24.190825+00:00",
    "employee_code": "POOW",
    "trace_ids": "",
    "location": "location1-backend-somewhere",
    "span_id": "BE.function2.deletebefore.list",
    "details": "{\"yyyymmdd\":\"20260129\",\"files\":[\"file_0041.mp3\"]}",
    "_bulk_data": null,
    "_bulk_data_error": null
}
...
```

ここで重要なのは、**エンティティ間の関係が暗黙的である**ということです。関係性は `details` フィールド内のJSONに埋もれており、`trace_ids`、`employee_code`、`span_id` の各カラムに散在しています。ファイルのライフサイクルを追跡するには、これらすべての次元を横断的に結合する必要があります。これはまさにグラフデータベースが得意とする処理ですが、本ハンズオンではグラフDBなしでこれを実現します。



## (2) 課題：複数の次元を1つのフィールドに埋め込む

### なぜグラフデータベースを使わないのか？

Neo4j [1] や TigerGraph [2] のようなグラフネイティブソリューションは、多次元の関係データを扱うために設計されています。ノード（ファイル、従業員、セッション）とエッジ（それらの間の関係）をネイティブに表現できます。しかし：

- **追加インフラが必要** — 別途データベースサーバー、新しいクエリ言語（Cypher [6]、GSQL）、運用コストが発生する
- 多くのチームはこれらのログを格納する**既存のリレーショナルデータベース**（PostgreSQL、MySQL、SQLite）を既に持っている
- 数千〜数百万行規模の探索的分析では、グラフDBの導入コストは正当化しにくい

### なぜ可視化ツールを直接使わないのか？

Graphviz [5]（dot記法、メトロマップ）や Mermaid [4] はグラフ描画に優れたツールですが、**構造化された入力**が必要です。生のログデータは、可視化の前にパース・抽出のステージを経る必要があります。

### アプローチ：graph-in-flat-DB

考え方はシンプルです：

1. 各ログエントリから全てのユニーク識別子（ファイル名、セッションID等）を**抽出**する
2. 各識別子を短い衝突耐性のある文字列に**ハッシュ化**する（MD5の先頭14文字）
3. 1つのログエントリに対応する全ハッシュをスペース区切りで `trace_ids` フィールドに**格納**する
4. 再帰SQLを用いて関係の分割・結合・追跡を**クエリ**する
5. ユニークな共起パターンをメトロマップグラフとしてエクスポートし**可視化**する

これにより、既存のリレーショナルデータベースを活用しつつ、エンティティ間の多次元的な関係性を捕捉できます。最終的な目標は2つです：

1. **ライフサイクル追跡：** 1つのファイルが処理ステージ、セッション、従業員を跨いでどう流れるかを追跡する
2. **パターン発見：** Python [7] やRのグラフ分析ライブラリを用いて、中心性、クラスタリング、異常検知などのグラフ分析を行う — DBから抽出したトレースIDのバッチを入力として使用する



## (3) 解法1：グラフノードの定義

従来のグラフモデルでは、明示的なノードタイプとエッジタイプを定義します。本アプローチでは、よりシンプルなスタンスを取ります：

> **1つのログエントリが1つのノード。そのエントリに含まれるユニーク識別子は、同じ識別子を共有する他のノードへの「リンク」である。**

つまり：

- **ファイル**（`file_0008.mp3`）はそれ自体ノードではなく、`file_0008.mp3` に言及する全てのログエントリが、同じく言及する他の全てのログエントリと接続される
- **従業員**（`POOW`）も同様に、その従業員に関連する全てのログエントリを接続する
- **セッション**は、同一セッションIDを共有する全てのログエントリを接続する

「エッジ」は暗黙的です：2つのログエントリが `trace_ids` フィールドで少なくとも1つの識別子を共有していれば、それらは接続されています。共有する識別子が多いほど、接続は強くなります。これは本質的に**ハイパーグラフ** [8] をフラットなリレーショナルスキーマに射影したものです。


## (4) 解法2：ハッシュベースの識別

複数の識別子を1つのテキストフィールドに効率的に格納するため、各ユニーク値（ファイル名、セッションID等）をMD5でハッシュ化し、先頭14文字の16進数に切り詰めます：

- `md5("file_0001.mp3")` → `e2c569be17396eca2a2e3c11578123ed` → **`e2c569be17396e`**

**なぜ14文字か？** 14桁の16進数における衝突確率は約 1/16¹⁴ ≈ 1/7.2×10¹⁶ です。数千〜数百万のユニーク識別子を持つデータセットでは、実質的にゼロと見なせます。切り詰めにより格納領域を節約し、`trace_ids` フィールドの可読性も向上します。

ハッシュ化は [`runme-js-to-db.js`](md-graph-in-flat-db/runme-js-to-db.js) スクリプトによりデータベース初期化時に実行されます。各ログエントリの `details` JSONフィールドをパースし、ファイルID、セッションID、その他のユニーク値を抽出し、各値をハッシュ化して、スペース区切りで `trace_ids` フィールドに連結します。

結果として得られるスキーマは以下の通りです：

```
span_id                             | details (with embedded md5 hash ids for files, sessions, etc.) | trace_ids (md5 of file names, session ids, etc.)
BE.function2.get                     | file_0051.mp3, file_0086.mp3, file_0087.mp3, fil | 4931d3afc6e672 4d22a4251f1d1c 4eb6bf15031600 2479dde7ed882b 807b6b1724d30e
BE.function3.post.done               | file_0024                                        | f90197d492238b
BE.function3.files.after             | file_0024                                        | f90197d492238b
BE.function3.files.before            | file_0024                                        | f90197d492238b
```

各行の `trace_ids` フィールドは、14文字のハッシュをスペースで区切ったリストです。5つのファイルを返す `BE.function2.get` 呼び出しには5つのハッシュが含まれます。1つのファイルを処理する `BE.function3.post.done` 呼び出しには1つのハッシュのみが含まれます。このエンコーディングが、後続の全てのクエリの基盤となります。



## (5) 解法3：再帰SQLによるクエリ

識別子が `trace_ids` フィールドに埋め込まれたので、次はそれらを抽出・分析するSQLクエリが必要です。SQLiteの再帰共通テーブル式（Recursive CTE）[9] のサポートにより、アプリケーションコードなしでこれが可能になります。

### 準備

クエリを実行する前に、ローカルのSQLiteデータベースをセットアップします：

1. **Node.js**（v16以降）がインストールされていることを確認
2. `better-sqlite3` パッケージをインストール：`npm install better-sqlite3`
3. 匿名化済みデータセットからデータベースを初期化：

これにより、匿名化済みの全ログエントリと事前計算された `trace_ids` を含む `BusinessLog` テーブルを持つ `260206.db` SQLiteファイルが作成されます。

以下のサブセクションでは、単純な閲覧から完全なライフサイクル追跡まで、段階的にクエリの複雑さを積み上げていきます。

```bash
node runme-js-to-db.js --db 260206.db --command initdb --input 260206-dataset-anonymized.json 
```

### (5-1) ステップ1：trace_idsの確認（クエリ：`260206-Q1.sql`）

最もシンプルなクエリとして、1つ以上のトレースIDを持つ全ログエントリを取得します。これにより、データの量と `trace_ids` フィールドの構造を把握できます：

```sql
-- Simple view of all entries with trace_ids
SELECT 
    id,
    span_id,
    trace_ids,
    created_at
FROM BusinessLog
WHERE trace_ids IS NOT NULL AND trace_ids != ''
ORDER BY created_at;
```

出力を見ると、単一のハッシュを持つエントリ（例：`c71b345135dfd9` — 1つのファイル）もあれば、スペースで区切られた複数のハッシュを持つエントリ（例：14個のファイルハッシュを持つ `BE.function2.get`）もあることが分かります。複数値のエントリこそが関係性をエンコードしているものです — 1回のAPI呼び出しで「一緒に見られた」ファイル群を示しています。

```bash
sqlite3 260206.db < 260206-Q1.sql
378739|BE.function2.get|c71b345135dfd9|2026-01-20T07:34:30.695348+00:00
378741|BE.function2.get|c71b345135dfd9|2026-01-20T07:35:01.299873+00:00
378752|BE.function3.files.before|73e39054576368|2026-01-20T07:39:33.745497+00:00
378753|BE.function3.files.after|73e39054576368|2026-01-20T07:39:33.763494+00:00
378754|BE.function3.post.kakutei|73e39054576368|2026-01-20T07:39:33.826545+00:00
378755|BE.function3.post.done|73e39054576368|2026-01-20T07:39:35.131301+00:00
378772|BE.function2.get|e844141f43e3f7 ae7a40b9fa4a21|2026-01-20T07:46:13.195698+00:00
378775|BE.function2.get|c71b345135dfd9|2026-01-20T07:47:20.786875+00:00
378777|BE.function2.get|c71b345135dfd9|2026-01-20T07:47:40.710533+00:00
378779|BE.function2.get|e844141f43e3f7 ae7a40b9fa4a21|2026-01-20T07:47:56.210949+00:00
378780|BE.function2.get|c71b345135dfd9|2026-01-20T07:48:08.105727+00:00
378784|BE.function2.get|e844141f43e3f7 ae7a40b9fa4a21|2026-01-20T07:49:45.603084+00:00
378793|BE.function2.get|61070dafab8640 89c1323e7bf0a6 009f2d25f54896 ff3b3983d02faa ecfeb81a6c3c26 aa725923904dda 2b010e839d0848 3db16a2410a415 ec25183f3acb1
8 618a8690c39cc5 2f03d8f0f21d0a 47f541dc92bbe0 73e39054576368 c67b53076a53f7|2026-01-20T07:53:49.812955+00:00
378817|BE.function2.get|e844141f43e3f7 ae7a40b9fa4a21|2026-01-20T08:04:22.870693+00:00
...
```

### (5-2) ステップ2：スペース区切りのtrace_idsを行に分割（クエリ：`260206-Q2.sql`）

個々のトレースIDを扱うには、スペース区切りの文字列を個別の行に分割する必要があります。SQLiteにはビルトインの `STRING_SPLIT` 関数がありません（PostgreSQLの `STRING_TO_TABLE` やSQL Serverの `STRING_SPLIT` とは異なります）。そこで、1つずつトークンを剥がしていく**再帰CTE** [9] を使用します：

```sql
-- Split space-delimited trace_ids into individual rows
WITH RECURSIVE split_trace(id, span_id, created_at, trace_id, rest) AS (
    -- Base case: get the first trace_id and the rest of the string
    SELECT 
        id,
        span_id,
        created_at,
        CASE 
            WHEN INSTR(trace_ids || ' ', ' ') > 0 
            THEN SUBSTR(trace_ids || ' ', 1, INSTR(trace_ids || ' ', ' ') - 1)
            ELSE trace_ids
        END,
        CASE 
            WHEN INSTR(trace_ids || ' ', ' ') > 0 
            THEN SUBSTR(trace_ids || ' ', INSTR(trace_ids || ' ', ' ') + 1)
            ELSE ''
        END
    FROM BusinessLog
    WHERE trace_ids IS NOT NULL AND trace_ids != ''
    
    UNION ALL
    
    -- Recursive case: continue splitting the rest
    SELECT 
        id,
        span_id,
        created_at,
        CASE 
            WHEN INSTR(rest, ' ') > 0 
            THEN SUBSTR(rest, 1, INSTR(rest, ' ') - 1)
            ELSE rest
        END,
        CASE 
            WHEN INSTR(rest, ' ') > 0 
            THEN SUBSTR(rest, INSTR(rest, ' ') + 1)
            ELSE ''
        END
    FROM split_trace
    WHERE rest != ''
)
SELECT id, span_id, created_at, trace_id
FROM split_trace
WHERE trace_id != ''
ORDER BY created_at;
```

実行して以下のように出力される：

```bash
sqlite3 260206.db < 260206-Q2.sql
378739|BE.function2.get|2026-01-20T07:34:30.695348+00:00|c71b345135dfd9
378741|BE.function2.get|2026-01-20T07:35:01.299873+00:00|c71b345135dfd9
378752|BE.function3.files.before|2026-01-20T07:39:33.745497+00:00|73e39054576368
378753|BE.function3.files.after|2026-01-20T07:39:33.763494+00:00|73e39054576368
378754|BE.function3.post.kakutei|2026-01-20T07:39:33.826545+00:00|73e39054576368
378755|BE.function3.post.done|2026-01-20T07:39:35.131301+00:00|73e39054576368
378772|BE.function2.get|2026-01-20T07:46:13.195698+00:00|e844141f43e3f7
378772|BE.function2.get|2026-01-20T07:46:13.195698+00:00|ae7a40b9fa4a21
378775|BE.function2.get|2026-01-20T07:47:20.786875+00:00|c71b345135dfd9
378777|BE.function2.get|2026-01-20T07:47:40.710533+00:00|c71b345135dfd9
378779|BE.function2.get|2026-01-20T07:47:56.210949+00:00|e844141f43e3f7
378779|BE.function2.get|2026-01-20T07:47:56.210949+00:00|ae7a40b9fa4a21
378780|BE.function2.get|2026-01-20T07:48:08.105727+00:00|c71b345135dfd9
378784|BE.function2.get|2026-01-20T07:49:45.603084+00:00|e844141f43e3f7
378784|BE.function2.get|2026-01-20T07:49:45.603084+00:00|ae7a40b9fa4a21
378793|BE.function2.get|2026-01-20T07:53:49.812955+00:00|61070dafab8640
378793|BE.function2.get|2026-01-20T07:53:49.812955+00:00|89c1323e7bf0a6
378793|BE.function2.get|2026-01-20T07:53:49.812955+00:00|009f2d25f54896
378793|BE.function2.get|2026-01-20T07:53:49.812955+00:00|ff3b3983d02faa
```

再帰CTEは2つのフェーズで動作します：

1. **ベースケース：** `BusinessLog` の各行について、最初のトークン（最初のスペースまでの部分）を抽出し、残りを保持する
2. **再帰ケース：** 残りの部分から次のトークンを抽出し、空になるまで繰り返す

結果として、`378772|BE.function2.get|e844141f43e3f7 ae7a40b9fa4a21|...` のような1行が、各トレースIDごとに2行に展開されます。この「展開」されたビューが、後続の全ての分析（集計、グルーピング、結合、追跡）の基盤となります。


### (5-3) ステップ3：ユニークなトレースIDの抽出（クエリ：`260206-Q3.sql`）

分割CTEを基に、`SELECT DISTINCT` を使ってデータセット内の全ユニーク識別子のセットを抽出できます：

```sql
-- Get all unique trace_ids
WITH RECURSIVE split_trace(id, span_id, created_at, trace_id, rest) AS (
    SELECT 
        id,
        span_id,
        created_at,
        CASE 
            WHEN INSTR(trace_ids || ' ', ' ') > 0 
            THEN SUBSTR(trace_ids || ' ', 1, INSTR(trace_ids || ' ', ' ') - 1)
            ELSE trace_ids
        END,
        CASE 
            WHEN INSTR(trace_ids || ' ', ' ') > 0 
            THEN SUBSTR(trace_ids || ' ', INSTR(trace_ids || ' ', ' ') + 1)
            ELSE ''
        END
    FROM BusinessLog
    WHERE trace_ids IS NOT NULL AND trace_ids != ''
    
    UNION ALL
    
    SELECT 
        id,
        span_id,
        created_at,
        CASE 
            WHEN INSTR(rest, ' ') > 0 
            THEN SUBSTR(rest, 1, INSTR(rest, ' ') - 1)
            ELSE rest
        END,
        CASE 
            WHEN INSTR(rest, ' ') > 0 
            THEN SUBSTR(rest, INSTR(rest, ' ') + 1)
            ELSE ''
        END
    FROM split_trace
    WHERE rest != ''
)
SELECT DISTINCT trace_id
FROM split_trace
WHERE trace_id != ''
ORDER BY trace_id;
```

実装する：

```bash
sqlite3 260206.db < 260206-Q3.sql | more
009f2d25f54896
03951b21d95b3a
046ffc14874a96
05cbaf5a147acf
0601fa72f290d5
0676a877cacfbd
0677bce5c0013e
097cbcf89a0720
0b3761bed30f35
0cc1916a516854
0cda7748102fdb
0d792633305cd6
...
```

ユニークIDのフラットなリストはそれ自体ではあまり有用ではありません — どのIDがファイルを表し、どれがセッションを表すかは判別できません（その情報はハッシュ化の過程で意図的に失われています）。しかし、このリストは2つの重要な目的に役立ちます：

1. **カーディナリティの確認：** データセット内にユニークなエンティティがいくつ存在するか？ これによりグラフの複雑さを見積もれます。
2. **構成要素：** 分割CTEや追加の結合と組み合わせることで、次のステップの完全なライフサイクルクエリの基盤となります。



### (5-4) ステップ4：完全なライフサイクル追跡（クエリ：`260206-Q4.sql`）

これが集大成のクエリです。再帰的な分割と集約（`COUNT`、`MIN`、`MAX`）、そして結合を組み合わせて、全てのトレースIDの完全なタイムラインを出現頻度順に生成します：

```sql
-- Summary: count of entries per trace_id with timeline
WITH RECURSIVE split_trace(log_id, span_id, created_at, employee_code, trace_id, rest) AS (
    SELECT 
        id,
        span_id,
        created_at,
        employee_code,
        CASE 
            WHEN INSTR(trace_ids || ' ', ' ') > 0 
            THEN SUBSTR(trace_ids || ' ', 1, INSTR(trace_ids || ' ', ' ') - 1)
            ELSE trace_ids
        END,
        CASE 
            WHEN INSTR(trace_ids || ' ', ' ') > 0 
            THEN SUBSTR(trace_ids || ' ', INSTR(trace_ids || ' ', ' ') + 1)
            ELSE ''
        END
    FROM BusinessLog
    WHERE trace_ids IS NOT NULL AND trace_ids != ''
    
    UNION ALL
    
    SELECT 
        log_id,
        span_id,
        created_at,
        employee_code,
        CASE 
            WHEN INSTR(rest, ' ') > 0 
            THEN SUBSTR(rest, 1, INSTR(rest, ' ') - 1)
            ELSE rest
        END,
        CASE 
            WHEN INSTR(rest, ' ') > 0 
            THEN SUBSTR(rest, INSTR(rest, ' ') + 1)
            ELSE ''
        END
    FROM split_trace
    WHERE rest != ''
),
trace_entries AS (
    SELECT 
        trace_id,
        span_id,
        created_at,
        employee_code,
        log_id
    FROM split_trace
    WHERE trace_id != ''
),
trace_counts AS (
    SELECT 
        trace_id,
        COUNT(*) as entry_count,
        MIN(created_at) as first_seen,
        MAX(created_at) as last_seen
    FROM trace_entries
    GROUP BY trace_id
)
SELECT 
    e.trace_id AS hash_id,
    c.entry_count,
    e.span_id,
    e.created_at,
    e.employee_code
FROM trace_entries e
JOIN trace_counts c ON e.trace_id = c.trace_id
ORDER BY c.entry_count DESC, e.trace_id, e.created_at;
```


実行して以下のように出力される：

```bash
sqlite3 260206.db < 260206-Q4.sql | more
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:10:45.754536+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:10:48.282510+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:10:56.865942+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:11:41.030037+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:11:43.278977+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:11:46.350796+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-01-30T08:11:52.555053+00:00|POOW
c7a6ba3a2c7b1b|630|BE.file.new|2026-01-30T08:13:31.492183+00:00|POOW
c7a6ba3a2c7b1b|630|BE.file.linked|2026-01-30T08:13:31.553724+00:00|POOW
c7a6ba3a2c7b1b|630|function6.session.complete|2026-01-30T08:13:38.933844+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function1.post.file|2026-01-30T08:32:29.819139+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-02-01T23:56:30.500816+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-02-01T23:56:30.502671+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-02-01T23:56:30.502809+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-02-01T23:56:39.708242+00:00|POOW
c7a6ba3a2c7b1b|630|BE.function2.get|2026-02-01T23:56:41.229994+00:00|POOW
...
```

このクエリは3つの論理的なステージで構成されています：

1. **`split_trace`** — `trace_ids` を個別の行に分割する再帰CTE。`log_id`、`span_id`、`created_at`、`employee_code` を併せて運搬する
2. **`trace_entries`** — 空のトレースIDを除外するシンプルなフィルタ
3. **`trace_counts`** — 集約：各トレースIDを参照するログエントリの数、および最初と最後に観測された時刻

最後の `SELECT` でエントリとカウントを結合し、最も参照頻度の高いトレースIDから順に並んだ非正規化タイムラインを生成します。

出力カラムは以下の通りです：

| カラム | 説明 |
|--------|------|
| `hash_id` | ファイル名、セッションID、その他のユニーク識別子のMD5ハッシュ（14文字） |
| `entry_count` | このトレースIDを参照するログエントリの総数 |
| `span_id` | 処理ステージまたは関数名（例：`BE.function2.get`、`BE.file.new`） |
| `created_at` | ログエントリのタイムスタンプ（ISO 8601形式） |
| `employee_code` | 匿名化済みの従業員識別子 |


#### 出力の読み方

最も参照頻度の高いトレースID（`c7a6ba3a2c7b1b`、630エントリ）を時系列で読むと、明確なストーリーが見えます：

1. **ポーリングフェーズ**（`BE.function2.get` の繰り返し）：ユーザーがファイル一覧を複数回確認する
2. **アップロードフェーズ**（`BE.file.new` → `BE.file.linked`）：新規ファイルが作成され、リンクされる
3. **外部処理**（`function6.session.complete`）：外部サービス（例：文字起こし）が処理を完了する
4. **後処理**（`BE.function1.post.file`）：バックエンドがファイルを最終処理する
5. **後続のポーリング**（`BE.function2.get` の再度の繰り返し）：ファイルが以降のリスト取得で表示されるようになる

これは**1つのファイルのライフサイクル** — アップロードから外部処理を経て利用可能になるまで — をフラットなログデータからSQLのみで追跡したものです。




## (6) 解法4：メトロマップによる可視化

ライフサイクルクエリ（5-4）ではテキストベースのタイムラインが得られました。しかし、多数のトレースIDにわたるパターンを同時に発見するには、**視覚的な表現**が必要です。ここでは「メトロマップ」のメタファーを用います：

- 各**ユニークハッシュID**が「駅」
- **ユニークなハッシュIDの組み合わせ**（つまり、ログエントリの `trace_ids` の各固有値）が、それらの駅を通過する「路線」
- 頻繁に一緒に出現する駅は密に接続されている — 同じ操作で共起するエンティティを表す

これは本質的に、交通路線図として射影された**共起グラフ** [10] です。

### (6-1) SQLによる路線の抽出

以下のクエリは、2つ以上のハッシュを含む（つまり複数エンティティの関係を持つ）`trace_ids` の固有値を上位10件取得します：

```sql
-- Get unique trace_id lists that contain more than 1 id (space-delimited), top 10
SELECT DISTINCT trace_ids
FROM BusinessLog
WHERE trace_ids IS NOT NULL 
  AND trace_ids != '' 
  AND LENGTH(trace_ids) - LENGTH(REPLACE(trace_ids, ' ', '')) >= 1
ORDER BY trace_ids
LIMIT 10;
```

以下のような出力が得られる：

```bash
03951b21d95b3a 7d99009ec76167
03951b21d95b3a 7d99009ec76167 881b35c825cfef
03951b21d95b3a 881b35c825cfef
046ffc14874a96 843a36d3028503
046ffc14874a96 c3736083f514d9 2329edd4ec5a22 fcb338a9e961f7
046ffc14874a96 c3736083f514d9 d274b4f22ada77 fcb338a9e961f7 6d84c4164522d2 edcee1d5463a87 2329edd4ec5a22
046ffc14874a96 c3736083f514d9 fcb338a9e961f7 2329edd4ec5a22
046ffc14874a96 c3736083f514d9 fcb338a9e961f7 6d84c4164522d2 2329edd4ec5a22
046ffc14874a96 c3736083f514d9 fcb338a9e961f7 6d84c4164522d2 edcee1d5463a87 2329edd4ec5a22
046ffc14874a96 c3736083f514d9 fcb338a9e961f7 6d84c4164522d2 edcee1d5463a87 2329edd4ec5a22 d274b4f22ada77
```

### (6-2) Mermaidメトロマップ

これらの路線を Mermaid [4] の `graph LR` ダイアグラムに変換できます。各ユニークハッシュはラベル付きノード（可読性のため先頭6文字と末尾6文字を表示）となり、各路線は路線番号でラベル付けされた有向エッジの列になります：

```
graph LR
    %% Define stations (unique trace_ids) with short labels
    A["03951b<br/>...d95b3a"]
    B["7d9900<br/>...c76167"]
    C["881b35<br/>...25cfef"]
    D["046ffc<br/>...874a96"]
    E["843a36<br/>...028503"]
    F["c37360<br/>...f514d9"]
    G["2329ed<br/>...c5a22"]
    H["fcb338<br/>...961f7"]
    I["d274b4<br/>...ada77"]
    J["6d84c4<br/>...522d2"]
    K["edcee1<br/>...463a87"]

    %% Train 1: 03951b --> 7d9900
    A -->|line1| B

    %% Train 2: 03951b --> 7d9900 --> 881b35
    A -->|line2| B -->|line2| C

    %% Train 3: 03951b --> 881b35
    A -->|line3| C

    %% Train 4: 046ffc --> 843a36
    D -->|line4| E

    %% Train 5: 046ffc --> c37360 --> 2329ed --> fcb338
    D -->|line5| F -->|line5| G -->|line5| H

    %% Train 6: 046ffc --> c37360 --> d274b4 --> fcb338 --> 6d84c4 --> edcee1 --> 2329ed
    D -->|line6| F -->|line6| I -->|line6| H -->|line6| J -->|line6| K -->|line6| G

    %% Train 7: 046ffc --> c37360 --> fcb338 --> 2329ed
    D -->|line7| F -->|line7| H -->|line7| G

    %% Train 8: 046ffc --> c37360 --> fcb338 --> 6d84c4 --> 2329ed
    D -->|line8| F -->|line8| H -->|line8| J -->|line8| G

    %% Train 9: 046ffc --> c37360 --> fcb338 --> 6d84c4 --> edcee1 --> 2329ed
    D -->|line9| F -->|line9| H -->|line9| J -->|line9| K -->|line9| G

    %% Train 10: 046ffc --> c37360 --> fcb338 --> 6d84c4 --> edcee1 --> 2329ed --> d274b4
    D -->|line10| F -->|line10| H -->|line10| J -->|line10| K -->|line10| G -->|line10| I

    %% Styling - red cluster for 03951b group, blue cluster for 046ffc group
    style A fill:#e74c3c,color:#fff
    style B fill:#e74c3c,color:#fff
    style C fill:#e74c3c,color:#fff
    style D fill:#3498db,color:#fff
    style E fill:#3498db,color:#fff
    style F fill:#2ecc71,color:#fff
    style G fill:#f39c12,color:#fff
    style H fill:#9b59b6,color:#fff
    style I fill:#1abc9c,color:#fff
    style J fill:#e67e22,color:#fff
    style K fill:#34495e,color:#fff
```

結果＝画像は以下になります：

![](/images/README-sketch.png)


### (6-3) Graphvizによる代替表現

Graphviz [5] では `dot` 言語を使ってより簡潔な記法が可能です。各路線は `--` で駅を接続する1つのステートメントになります。レイアウトエンジンがノードの配置とエッジのルーティングを自動的に処理します：

```
graph G {
    line1 -- 03951b21d95b3a -- 7d99009ec76167
    line2 -- 03951b21d95b3a -- 7d99009ec76167 -- 881b35c825cfef
    line3 -- 03951b21d95b3a -- 881b35c825cfef
    line4 -- 046ffc14874a96 -- 843a36d3028503
    line5 -- 046ffc14874a96 -- c3736083f514d9 -- 2329edd4ec5a22 -- fcb338a9e961f7
    line6 -- 046ffc14874a96 -- c3736083f514d9 -- d274b4f22ada77 -- fcb338a9e961f7 -- 6d84c4164522d2 -- edcee1d5463a87 -- 2329edd4ec5a22
    line7 -- 046ffc14874a96 -- c3736083f514d9 -- fcb338a9e961f7 -- 2329edd4ec5a22
    line8 -- 046ffc14874a96 -- c3736083f514d9 -- fcb338a9e961f7 -- 6d84c4164522d2 -- 2329edd4ec5a22
    line9 -- 046ffc14874a96 -- c3736083f514d9 -- fcb338a9e961f7 -- 6d84c4164522d2 -- edcee1d5463a87 -- 2329edd4ec5a22
    line10 -- 046ffc14874a96 -- c3736083f514d9 -- fcb338a9e961f7 -- 6d84c4164522d2 -- edcee1d5463a87 -- 2329edd4ec5a22 -- d274b4f22ada77
}
```

結果は以下の通り：

![](/images/README-sketch-graphviz.png)


より大規模なデータセットでは、数百〜数千の路線をエクスポートし、NetworkX [7]（Python）や igraph [11]（R/Python）のようなライブラリを使って、中心性指標の計算、コミュニティ検出、異常検知などのグラフ分析をプログラム的に行うことができます。



---

## まとめ

| ステップ | 内容 | ツール |
|----------|------|--------|
| データ準備 | 識別子のハッシュ化、`trace_ids` への格納 | Node.js + better-sqlite3 |
| クエリ1 | trace_idsの生データ確認 | SQLite CLI |
| クエリ2 | 複数値フィールドを行に分割 | 再帰CTE |
| クエリ3 | ユニーク識別子の抽出 | `SELECT DISTINCT` |
| クエリ4 | 完全なライフサイクルタイムライン | 再帰CTE + JOIN + 集約 |
| 可視化 | メトロマップダイアグラム | Mermaid / Graphviz |

重要な知見は、**グラフ分析にグラフデータベースは必須ではない**ということです。関係性をスペース区切りのハッシュとして1つのフィールドにエンコードすることで、標準的なSQLでそれらの関係を抽出・分割・追跡でき、その結果を任意の可視化ツールやグラフ分析ツールに入力できます。

---

## 参考文献

[1] Neo4j Graph Database. https://neo4j.com/

[2] TigerGraph. https://www.tigergraph.com/

[3] SQLite. https://www.sqlite.org/

[4] Mermaid — Diagramming and charting tool. https://mermaid.js.org/

[5] Graphviz — Graph Visualization Software. https://graphviz.org/

[6] Francis, N., Green, A., Guagliardo, P., et al., "Cypher: An Evolving Query Language for Property Graphs," Proceedings of the 2018 International Conference on Management of Data (SIGMOD), pp. 1433–1445, 2018.

[7] Hagberg, A. A., Schult, D. A., Swart, P. J., "Exploring Network Structure, Dynamics, and Function using NetworkX," Proceedings of the 7th Python in Science Conference (SciPy), pp. 11–15, 2008. https://networkx.org/

[8] Berge, C., "Graphs and Hypergraphs," North-Holland Publishing Company, 1973.

[9] SQLite WITH clause (Common Table Expressions). https://www.sqlite.org/lang_with.html

[10] Newman, M. E. J., "Networks: An Introduction," Oxford University Press, 2010.

[11] Csardi, G., Nepusz, T., "The igraph software package for complex network research," InterJournal Complex Systems, 1695, 2006. https://igraph.org/