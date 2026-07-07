---
name: ingest
description: raw/ 配下の一次資料を取り込み、wiki/sources と wiki/concepts を更新する。新しい資料を追加した後や、未取り込みの資料をまとめて処理したいときに使う。
argument-hint: [raw/配下のファイルパス（省略時は未取り込みの全ファイル）]
---

`raw/` の一次資料を読み、`/CLAUDE.md` のスキーマに従って `wiki/` を更新する。

## 対象ファイルの決定

- `$ARGUMENTS` にパスが指定されていれば、それを対象にする（`raw/` 配下であることを確認する）。
- 指定がなければ `raw/` を走査し、対応する `wiki/sources/<slug>.md` がまだ存在しないファイルを全て対象にする。
- 対象が一つもなければ「取り込む新しい資料がありません」と報告して終了する。

## 手順（対象ファイルごとに）

1. `raw/<file>` を読む。
2. `wiki/sources/<slug>.md` を作成/更新する（slugはファイル名から）。内容は「資料の概要」「主張していること」「関連する `concepts/` ページ」の3点を短くまとめる。CLAUDE.mdの sources/*.md の項を参照。
3. 資料が扱っている概念（inotify, kqueue, fanotify 等）を特定し、該当する `wiki/concepts/<concept>.md` を作成または更新する。
   - 新規作成の場合は CLAUDE.md の concepts/*.md 構成（Overview / API・semantics / Limitations & gotchas / Platform notes / Related concepts / Sources）に従う。書くことがないセクションは省略してよい。
   - 既存ページを更新する場合は、矛盾する記述がないか確認し、frontmatterの `updated` と `sources` を更新する。
   - 既存の記述と矛盾する情報が見つかった場合は、上書きせずユーザーに報告してから判断を仰ぐ。
4. 新しいページを作成した場合は `wiki/index.md` の該当セクションにリンクを追加する。
5. `wiki/log.md` の先頭に日付見出し（無ければ追加）で今回の変更内容を1〜数行で追記する。

## 完了後の報告

処理した資料ごとに、作成/更新したページ一覧を簡潔に報告する。取り込み中に判断に迷った点（矛盾、曖昧な分類など）があれば併せて報告する。
