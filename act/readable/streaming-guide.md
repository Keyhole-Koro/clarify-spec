# Act Streaming Guide（Human View）

## まず覚えること

* `RunAct` は server-streaming
* イベントは `RunActEvent`
* `done` と `error` は排他
* `patch_ops` は `upsert` / `append_md` のみ

## 主要ケース

1. 通常回答
* `upsert` -> `append_md`... -> `done`

2. thinkingあり
* `stream_parts(thought=true)` -> `stream_parts(thought=false)` -> `patch_ops` -> `done`

3. 途中エラー
* 部分結果が表示済みでも `error` で終端（巻き戻しなし）

4. 同一 `request_id` 再送
* 二重実行防止（`ALREADY_EXISTS` またはキャッシュ結果）

## フロント実装の最低ルール

* thought と answer は別state
* `append_md` は追記のみ
* error時は `stage/retryable` で再試行UIを分岐

## 正本参照

* `act/specs/behavior/ai-streaming-cases.md`
* `act/specs/behavior/frontend-stream-integration.md`
* `act/specs/contracts/rpc-connect-schema.md`
