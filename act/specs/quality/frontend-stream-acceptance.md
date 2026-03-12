# Frontend 受け入れチェック（Act Stream）

対象:

* `features/knowledgeTree/store.ts`
* `features/actExplore/hooks/useActStream.ts`
* `features/graph/components/GraphCanvas.tsx`

## 1. Patch適用

* `upsert` で block が追加/更新される
* `append_md` で `contentMd` に追記される（置換しない）
* 未作成blockへの `append_md` を受けても破綻しない

## 2. Stream終端

* `done=true` 受信で loading が解除される
* `error` 受信で loading が解除され、通知が表示される
* `error.stage` と `error.retryable` をUI文言へ反映できる
* サーバ契約として `done` と `error` の二重終端を許可しない
* frontend は duplicate terminal を受けても最初の terminal のみ採用し、UI が壊れない
* `done` の後に `error` が来ても error UI を出さない
* `error` の後に `done` が来ても completed UI へ反転しない

## 3. 描画

* `ACT_ROOT` を起点に最低1ノード描画される
* 親子関係が edge として描画される
* ストリーム中の追記がリアルタイム反映される
* thought と answer が視覚的に混ざらない
* thought は node 本文へ混ざらない

## 4. 失敗耐性

* 途中失敗時でも既に描画済みノードは保持
* 再送してもストアが壊れない
* 同一ID `upsert` は上書き動作になる
* 再実行時に前回 request の thought buffer と混ざらない

## 5. ログ/デバッグ

* `traceId` 相当（なければ request id）を dev log へ残す
* 受信 event 数、patch_ops 数を計測可能にする
* duplicate terminal を無害化した場合は dev warning を残せる

## 6. Thinkthrough UX

* thought 表示の既定は OFF
* `thinking_config.include_thoughts=true` の request でのみ thought 表示を有効化できる
* 実行中は thought セクションを展開表示できる
* `done/error` 後は thought セクションが既定で折りたたまれる
* 終端後も request 単位で thought 内容を保持できる

## 7. Deep Research Fallback

* Deep Research fallback はエラー通知ではなく、軽いインライン状態表示として出る
* fallback 通知を出しても stream 継続中の UI を壊さない
* fallback 時も同一 request として扱われる
* fallback の詳細理由は dev console / log で追跡できる

## 8. Metadata Projection

* grounding metadata は本文下の `References` セクションに表示される
* tool metadata は既定で `Diagnostics` に寄せられ、本文主表示に混ざらない
* metadata は本文 state と分離保持される
* 未対応 metadata も store と dev console では追跡できる
