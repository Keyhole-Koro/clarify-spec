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

## 2. 異常系: モデル途中失敗

入力:

* 正常入力
* モデル呼び出しでタイムアウトを注入

期待:

* 途中まで `patch_ops` を受信していてもよい
* 最終イベントは `error`（`UNAVAILABLE` or `DEADLINE_EXCEEDED`）
* `done=true` は返らない

## 3. 異常系: バリデーション失敗

入力:

* `act_type=ACT_TYPE_UNSPECIFIED` または `user_message` 空

期待:

* 最初のイベントで `error=INVALID_ARGUMENT`
* `patch_ops` は空
* `done=true` は返らない

## 4. 追加チェック（全ケース共通）

* `done` と `error` が同時に来ない
* `PatchOp` が `upsert` / `append_md` のみ
* イベント終了後に追撃イベントが来ない
