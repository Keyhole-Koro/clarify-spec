# RunAct 実装仕様（Go API + ADK Worker）

## 目的

`RunAct` を `topic_id` 中心で実行し、Go API から ADK Worker を利用して安定ストリームを返す。

## スコープ / 非スコープ

* スコープ: request検証、ADK委譲、stream返却
* 非スコープ: Organize write path、UI描画

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `context/assembly/core.md`
* `context/assembly/bundle-schema.md`
* `act/act-api/specs/runtime/adk-service-integration.md`
* `act/act-adk-worker/specs/context/context-assembly-execution-profile.md`
* `act/act-api/specs/reliability/idempotency-policy.md`
* `act/act-api/specs/reliability/error-mapping-matrix.md`
* `context/model/topic-model.md`
* `act/act-api/specs/security/session-and-auth-boundary.md`

## 契約（I/O）

入力:

* `RunActRequest`（`topic_id`, `request_id` 必須）
* `Authorization`, Cookie sid/csrf

出力:

* `RunActEvent` stream
* 失敗時 `ErrorInfo { code, retryable, stage, trace_id, retry_after_ms? }`

## 正常フロー

1. `AUTHN`: Firebase token 検証
2. token claim から `token_uid` を確定（request `uid` は互換用）
3. `SID_VALIDATE`: sid検証
4. `CSRF_VALIDATE`: Double Submit 照合
5. `AUTHZ`: token_uid -> workspace membership -> topic access
6. `VALIDATE_REQUEST`: `topic_id`, `request_id`, 上限検証
7. request `uid` が存在する場合は `token_uid` と一致検証
8. 冪等判定 `(token_uid, workspace_id, request_id)`
9. Go API が ADK Worker へ実行委譲
10. ADK Worker が `ASSEMBLY_* -> GENERATE_WITH_MODEL -> NORMALIZE_PATCH_OPS`
11. Go API が `RunActEvent` へ変換して `EMIT_STREAM`
12. `FINALIZE`: `done=true` で終端

## 異常フロー（error/retryable/stage）

* topic欠落: `INVALID_ARGUMENT`, `retryable=false`, `stage=VALIDATE_REQUEST`
* topicアクセス不可: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* ADK worker接続失敗: `UNAVAILABLE`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* ADK worker timeout: `DEADLINE_EXCEEDED`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* assembly参照失敗: `UNAVAILABLE`, `retryable=true`, `stage=ASSEMBLY_RETRIEVE`
* request重複: `ALREADY_EXISTS`, `retryable=false`, `stage=VALIDATE_REQUEST`
* 返却形式不正: `INTERNAL`, `retryable=false`, `stage=NORMALIZE_PATCH_OPS`

## 数値パラメータ

* `request_timeout_ms=90000`
* `adk_worker_timeout_ms=60000`
* `adk_worker_retry_max=1`
* `patch_ops_max_per_run=400`
* `context_node_ids_max=120`
* `thought_flush_interval_ms=500`

## cancel 伝播仕様

* `act-api` は client disconnect または server deadline 検知時に、worker 呼び出し context を即時 cancel する
* worker 呼び出しには request 単位の親 context を使い、下流の model/tool/polling に同一 cancellation tree を伝播する
* cancel 後は新規 `RunActEvent` を生成しない
* 既送信イベントは有効のまま巻き戻さない
* client disconnect 起因では追加の `terminal.error` を返さない
* server 側ログには `trace_id`, `request_id`, `cancel_source` を残す
* `cancel_source` は最低でも `client_disconnect`, `server_deadline`, `upstream_cancel` を扱う
* 下流 SDK が cooperative cancel 非対応の場合は、最短 timeout と best-effort cleanup で停止を補助する

## stream 順序保証

* 初回の本文系 `append_md` より先に、対象 block の `upsert` を送る
* 同一 `RunActEvent` 内の基本順序は `thought -> answer -> upsert -> append_md -> metadata -> terminal` とする
* `append_md` は対象 block の `upsert` より先行しない
* `append_md` は空文字を送らない
* metadata は本文送出を block せず、answer より先に必須化しない
* `done/error` は最後の event にのみ現れ、終端後に追加 event を送らない
* node 本文 state の正本は `append_md` として扱えるよう、patch 生成順を安定化する

## model profile 切替表

* 既定 profile は `Flash` とする
* `research_config.use_deep_research=true` の場合のみ `Deep Research` を選択する
* `grounding_config` は profile と独立に付与し、`enabled=true` の場合は grounding を有効化する
* `research_config.use_deep_research!=true` の場合は `Flash` を使う
* `Deep Research` 実行が timeout / `UNAVAILABLE` / 連続 5xx で継続不能になった場合は、同一 request 内で `Flash` へ fallback する
* profile の最終決定は worker 側で行う

## 受け入れ条件（DoD）

* `topic_id` なし request は必ず拒否
* `RunAct` 契約を変えず ADK経由で実行できる
* `ErrorInfo.stage` が assembly/生成段階まで一貫
* `done/error` 排他が守られる
* client disconnect / deadline 時に cancel が worker 下流まで伝播する
* `append_md` が対応 block の `upsert` より先行しない

## 実装メモ（最小）

* Go API は契約ゲートウェイとして維持する
* ADK Worker は read-only実行に限定する
* `tree_id` は UI scope で扱い、正本キーには使わない
