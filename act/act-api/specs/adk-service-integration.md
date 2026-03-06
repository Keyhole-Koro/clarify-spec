# ADK Service Integration 仕様（Act API Go ↔ ADK Worker）

## 目的

Go製 `Act API` と ADK Worker の責務境界を固定し、`RunAct` 契約を維持したまま ADK を導入する。

## スコープ / 非スコープ

* スコープ: サービス分割、I/O、ストリーミング変換、失敗時挙動
* 非スコープ: ADK内部プロンプト文面、Organize write path

## 前提 / 参照

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/context/core.md`
* `act/specs/context/implementation.md`
* `act/act-api/specs/runact-implementation.md`

## 構成

* `act-api`（Cloud Run / Go / connect-go）
  * AuthN/AuthZ、request検証、idempotency、RunAct stream公開
* `act-adk-worker`（Cloud Run / Python / Google ADK）
  * Context Assembly、tool呼び出し、モデル実行、thought/answer生成

## 契約（I/O）

### act-api -> adk-worker 入力

| 項目 | 内容 |
| --- | --- |
| `trace_id` | Run単位の追跡ID |
| `topic_id` | 知識正本キー |
| `workspace_id` | 境界検証済みID |
| `uid` | 認証済みユーザー |
| `request_id` | 冪等キー |
| `user_message` | 入力文 |
| `selected_node_ids` | UI選択文脈 |
| `llm/grounding/thinking/research config` | 実行設定 |

### adk-worker -> act-api 出力

| 項目 | 内容 |
| --- | --- |
| `stream_parts` | thought/answer増分 |
| `patch_ops` | `upsert` / `append_md` のみ |
| `usage` | token/latency指標 |
| `terminal` | done or error |

## 正常フロー

1. Go `RunAct` が認証/認可/入力検証を完了
2. Go が ADK Worker へ実行要求を送信
3. ADK Worker が Context Assembly 実行
4. ADK Worker がモデル/ツール実行をストリーム生成
5. Go が `RunActEvent` へ変換してクライアントへ返却
6. done で終端

## 異常フロー（error/retryable/stage）

* ADK worker 到達不可: `UNAVAILABLE`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* ADK worker timeout: `DEADLINE_EXCEEDED`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* ADK返却形式不正: `INTERNAL`, `retryable=false`, `stage=NORMALIZE_PATCH_OPS`
* ADK側 assembly失敗: `ASSEMBLY_*` へマッピングして返却

## 数値パラメータ

* `act-api -> adk-worker timeout`: 60s
* `act-api total request timeout`: 90s
* `act-api retry to adk-worker`: 1回（idempotent時のみ）

## 受け入れ条件（DoD）

* `RunAct` 契約を変えずに ADK 導入できる
* thought/answer/patch の順序保証がある
* ADK障害時のエラーコードと stage が一貫する

## 実装メモ（最小）

* `act-api` は stateless を維持
* ADK Worker は Firestore/GCS write を行わない
* Organize は従来どおり write path を担当
