---
title: "ブランチ間のコミット転送とチェリーピック：develop から test へ確実に反映する"
emoji: "🍒"
type: "tech"
topics: ["git", "github", "cherrypick", "workflow"]
published: true
---

# ブランチ間のコミット転送とチェリーピック：develop から test へ確実に反映する

**所要時間：** 読了 約10分  
**前提条件：** 日常的なGit操作（ブランチ、cherry-pick、リモート）

## 概要

同じ変更を受け取る必要がある長期ブランチを2本持つチームは少なくありません。私たちの場合、PRは `develop` に対して作成され、マージ後に同じ変更を `test` にも反映する必要があります。変更ごとに2つ目のPRを作りたくはなく、かつ、PRが触れたものについては両ブランチが同じコードになっていてほしいのです。

`git cherry-pick` は自然な解に見えますし、実際に答えの大半を占めます。しかし実務では隙間が残ります。2つのブランチは履歴が異なるため、チェリーピックしたファイルが `develop` のファイルと細かく食い違うことがあるのです。本記事では、PRをコミット単位で再適用したうえで、最後に明示的な比較ステップでその隙間を埋める手順を紹介します。

## 目次

1. **状況と、単純なcherry-pickでは足りない理由**
2. **手順：** 再適用、競合解消、整合、検証、プッシュ
3. **スクリプト**
4. **実際の実行結果（数値）**
5. **注意点とポリシー上の選択**

## 1. 状況

```
develop  ---o---o---[PRマージ]---
test     ---o---x---x---o---
```

`test` には `develop` にないコミット（後から昇格させる機能開発、環境固有の調整、診断用コードなど）があります。それでも両ブランチはほとんどのファイルを共有しています。PRが、`test` 側でも変更されているファイルに触れている場合、PRを `test` に再適用すると次の3通りのことが起こり得ます。

1. パッチがきれいに適用され、結果が `develop` の版と一致する。これが理想的なケースです。
2. パッチが競合する。Gitが止まるので、何を残すか判断する必要があります。
3. パッチはきれいに適用されるが、**結果はそれでも `develop` と異なる。** 同じファイルの別の箇所に `test` 固有の変更があるためです。Gitは何も警告しないので、誰も気づきません。

危険なのは3番目です。何も失敗しないのに、`test` 上のファイルは、古い `test` の版でも `develop` の版でもない状態になります。こうした静かな差異が積み重なると、「`test` は `develop` でレビューしたものを本当に動かしているのか」という問いに答えられなくなります。

そこで手順は2部構成にしています。cherry-pickは履歴を運びます。最後のファイル単位の比較は結果を保証します。

## 2. 手順

1. **`test` から作業ブランチを作る。**
   すべてをfetchし、古いローカルの `test` ではなく、リモートの `test` から分岐します。

2. **PRのコミットを古い順に再適用する。**
   マージコミットではなく、PR自体のコミット一覧を取得し、作業ブランチに1つずつcherry-pickします。`-x` を付けると元のSHAが各コミットメッセージに記録され、後の監査が格段に楽になります。PRがスカッシュマージされていても、元のコミットはPRのref（GitHubでは `refs/pull/<n>/head`）から辿れます。

3. **競合はPR側のコミットを優先して解消する。**
   cherry-pick中の `--theirs` は、適用しようとしているコミットを指します。これはあえて手を抜く選択です。間違いはステップ4で修復されるため、途中状態の手作業マージに労力をかける意味はありません。

4. **最終整合：触れたすべてのファイルを `develop` と比較する。**
   PRが触れたファイルの集合（全コミットの和集合）を集めます。各ファイルについて、作業ブランチと `develop` をdiffし、差異が少しでもあれば、そのファイルを `develop` の版で置き換えます（`git checkout origin/develop -- <file>`）。PRがファイルを削除していた場合は削除します。結果は、明示的な追加の1コミットとしてコミットします。

   ここでのポリシーは **「差異が残る場合は develop が優先」** です。これは技術的な細部ではなく判断であり、注意点の節で再度触れます。

5. **検証する。**
   比較を再実行します。触れたすべてのファイルが `develop` とバイト単位で一致していなければなりません。そうでなければ手順のバグなので、そこで止めます。

6. **プッシュする。**
   作業ブランチを `test` にプッシュし（`git push origin sync/<name>:test`。`test` から分岐したためfast-forwardです）、リモートの `test` のHEADが想定したコミットであることを確認します。

ステップ4の追加コミットは、監査記録も兼ねます。そのdiffを見れば何がずれていたかが正確に分かり、ステップ4でコミットが発生しないのが最良の結果です。

## 3. スクリプト

手順をコンパクトにまとめたものです。競合するファイル、静かにずれるファイル、削除されたファイルを含む使い捨てリポジトリで動作を確認しています。ブランチ名やエラー処理はご自身の環境に合わせてください。

```bash
#!/usr/bin/env bash
# usage: sync-pr.sh <base> <head> <name>
#   base..head : PRのコミット範囲。作業ブランチは sync/<name>
set -euo pipefail

SRC=origin/develop
DST=origin/test
BASE=$1; HEAD=$2; WORK=sync/$3

git fetch origin
git switch -c "$WORK" "$DST"

# 1. PRのコミットを古い順に1つずつ再適用
for c in $(git rev-list --reverse --no-merges "$BASE..$HEAD"); do
  if ! git cherry-pick -x "$c"; then
    # 2. 競合時はPR側を採用。誤りはステップ3で修復される
    git diff --name-only --diff-filter=U | while IFS= read -r f; do
      git checkout --theirs -- "$f" && git add -- "$f"
    done
    git cherry-pick --continue --no-edit || git cherry-pick --skip
  fi
done

# 3. 最終整合：PRが触れたファイルはすべて develop の版と一致させる
git diff --name-only --no-renames "$BASE" "$HEAD" | while IFS= read -r f; do
  if git cat-file -e "$SRC:$f" 2>/dev/null; then
    if ! git diff --quiet "$SRC" -- "$f"; then
      echo "drift: $f -> taking $SRC version"
      git checkout "$SRC" -- "$f"
    fi
  elif [ -e "$f" ]; then
    echo "drift: $f was deleted on develop -> removing"
    git rm -q -- "$f"
  fi
done
git diff --cached --quiet || git commit -m "sync: align PR files with ${SRC#origin/}"

# 4. 検証：触れたファイルに差異が残っていないこと
git diff --name-only --no-renames "$BASE" "$HEAD" | while IFS= read -r f; do
  git diff --quiet "$SRC" -- "$f" || { echo "STILL DIFFERENT: $f"; exit 1; }
done
echo "OK. Review, then: git push origin HEAD:${DST#origin/}"
```

重要な細部がいくつかあります。

- `--no-renames` を付けると、リネームの旧パスと新パスの両方が列挙され、両方が整合されます。
- 競合解消後に空になったcherry-pick（変更がすでに `test` に存在する場合）は、失敗ではなくスキップとして扱います。
- スクリプトはプッシュしません。共有ブランチへ直接プッシュするかどうかは、出力を見たうえで判断すべきだからです。

## 4. 実際の実行結果（数値）

この手順は、2つのサービス（GoバックエンドとDjangoバックエンド）のログ出力の挙動を変更したPRに対して使用しました。その実行結果は以下のとおりです。

| 項目 | 値 |
|---|---|
| PRのコミット数 | 3 |
| PRが触れたファイル数 | 11 |
| cherry-pickの競合 | 1（`test` が独自に診断ログを追加していた関数） |
| 3つのcherry-pick後も `develop` と異なったファイル | 11個中4個 |
| `test` に追加されたコミット | 4（再適用3 + 整合1） |
| 最終ステップ後に `develop` と一致したファイル | 11個中11個 |

教訓となるのは、ずれていた4ファイルです。そのどれも競合していませんでした。4つともきれいにcherry-pickされ、Gitの出力には問題を示すものが何もありませんでした。原因は、同じファイル内の無関係な `test` 固有の変更でした：追加の認証機能、デバッグ用の `print` 文、異なっていたimport、設定内のコネクションプールの調整です。最終比較がなければ、これら4ファイルは正しそうな見た目のまま、誤った状態で反映されていたはずです。

一方、競合した1ファイルは問題になりませんでした。PR側を採用して解消した時点ですでに `develop` と一致しており、ずれたファイルには現れませんでした。

## 5. 注意点とポリシー上の選択

**「develop 優先」は、それらのファイル内の `test` 固有の変更を捨てることになります。** ずれていた4ファイルでは、PRの行だけでなくファイル全体が置き換えられました。`test` 固有の変更も含まれます。私たちのポリシーはこれを許容します。目的は、PRが触れたものについて `test` が `develop` を鏡のように写すことであり、そうしたファイル内の `test` 固有コードは破棄してよいと判断したためです。`test` に残すべき変更がある場合は、このステップを変更してください。プッシュ前に整合のdiffを表示して人がレビューするか、`test` 固有のハンクを後続のコミットで再適用します。

**整合コミットが確認すべき場所です。** それが大きい、あるいは意外な内容であれば、ブランチが想定以上に乖離していることを示しています。これは有用な情報です。

**直接プッシュかPRか。** 変更は `develop` ですでにレビュー済みなので、`test` へはPRなしでプッシュしています。ブランチ保護でPRが必須の場合は、作業ブランチをプッシュしてPRを作成してください。手順自体は変わりません。

**検証を省略しないでください。** 安価なループ1つで、「cherry-pickした」を「ファイルが一致している」に変えられる唯一のステップです。

## まとめ

cherry-pickはコミットを運びますが、結果が元のブランチと一致することは保証しません。PRをコミット単位で再適用して履歴を自然に保ち、競合は後の手順で直るので手を抜いて解消し、そのうえで触れたすべてのファイルを `develop` とdiffして、差異が残る場合は `develop` の版を採用します。検証してからプッシュし、整合コミットを監査記録として残します。

## 参考文献

[1] Git documentation, `git-cherry-pick`. https://git-scm.com/docs/git-cherry-pick

[2] Git documentation, `git-checkout`（コミットからのパス指定チェックアウト）. https://git-scm.com/docs/git-checkout

[3] GitHub Docs, "Checking out pull requests locally"（`refs/pull/<n>/head`）. https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/checking-out-pull-requests-locally
