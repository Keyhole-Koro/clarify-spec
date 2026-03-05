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
* `done` と `error` の二重終端を許可しない

## 3. 描画

* `ACT_ROOT` を起点に最低1ノード描画される
* 親子関係が edge として描画される
* ストリーム中の追記がリアルタイム反映される

## 4. 失敗耐性

* 途中失敗時でも既に描画済みノードは保持
* 再送してもストアが壊れない
* 同一ID `upsert` は上書き動作になる

## 5. ログ/デバッグ

* `traceId` 相当（なければ request id）を dev log へ残す
* 受信 event 数、patch_ops 数を計測可能にする
