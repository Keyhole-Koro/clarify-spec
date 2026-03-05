# Act E2E テスト計画（最小3本）

対象: `ActService.RunAct`（Connect streaming）

## 1. 正常系: Stream成功

入力:

* `act_type=ACT_TYPE_EXPLORE`
* 有効な `tree_id`, `workspace_id`, `uid`
* `user_message` あり

期待:

* `patch_ops` を1回以上受信
* `upsert` に `ACT_ROOT` が含まれる
* 最終イベントは `done=true`
* `error` は発生しない

## 1.1 thought stream: 増分思考表示

入力:

* `thinking_config.include_thoughts=true`
* `act_type=ACT_TYPE_EXPLORE`

期待:

* `stream_parts[].thought=true` を1件以上受信
* `stream_parts[].thought=false`（回答）も受信
* 最終イベントは `done=true`

## 2. 異常系: モデル途中失敗

入力:

* 正常入力
* モデル呼び出しでタイムアウトを注入

期待:

* 途中まで `patch_ops` を受信していてもよい
* 最終イベントは `error`（`UNAVAILABLE` or `DEADLINE_EXCEEDED`）
* `done=true` は返らない

## 2.1 Vertex API エラーマッピング

入力:

* Vertex API 側で 429 / 503 / timeout を注入

期待:

* 429/503 は `UNAVAILABLE` 相当で返却
* timeout は `DEADLINE_EXCEEDED` 相当で返却
* 返却ログに `traceId` が出る

## 3. 異常系: バリデーション失敗

入力:

* `act_type=ACT_TYPE_UNSPECIFIED` または `user_message` 空

期待:

* 最初のイベントで `error=INVALID_ARGUMENT`
* `patch_ops` は空
* `done=true` は返らない

## 3.1 異常系: 重複 request_id

入力:

* 同一 `uid/workspace_id/request_id` で同じ RunAct を二重送信

期待:

* 2本目は `error=ALREADY_EXISTS`
* 2本目で新しい `patch_ops` は流れない
* 1本目の stream は継続または終端まで完了する

## 4. 追加チェック（全ケース共通）

* `done` と `error` が同時に来ない
* `PatchOp` が `upsert` / `append_md` のみ
* イベント終了後に追撃イベントが来ない
* thoughtの有無が `thinking_config` と整合する
* `error` 時は `retryable` / `stage` / `trace_id` が設定される
