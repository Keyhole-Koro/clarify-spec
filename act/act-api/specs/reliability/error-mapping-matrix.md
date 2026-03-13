# Error Mapping Matrix（Act API）

## 目的

`ErrorInfo` の `code/stage/retryable/retry_after_ms` を最終固定し、frontend挙動を安定化する。

## スコープ / 非スコープ

* スコープ: RunAct と招待系APIの共通マッピング
* 非スコープ: 内部スタックトレース公開

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runtime/runact-implementation.md`
* `act/act-api/specs/workspace/invite-link-api-contract.md`

## 契約（I/O）

出力:

* `ErrorInfo.code`（必須）
* `ErrorInfo.stage`（必須）
* `ErrorInfo.retryable`（必須）
* `ErrorInfo.trace_id`（必須）
* `ErrorInfo.retry_after_ms`（必要時）

## マッピング表（固定）

| source condition | code | stage | retryable | retry_after_ms |
| --- | --- | --- | --- | --- |
| token invalid | `UNAUTHENTICATED` | `AUTHN` | false | - |
| provider not google.com | `UNAUTHENTICATED` | `AUTHN` | false | - |
| workspace/topic access denied | `PERMISSION_DENIED` | `AUTHZ` | false | - |
| request validation error | `INVALID_ARGUMENT` | `VALIDATE_REQUEST` | false | - |
| csrf mismatch | `PERMISSION_DENIED` | `CSRF_VALIDATE` | false | - |
| redis unavailable | `UNAVAILABLE` | `SID_VALIDATE` | true | 3000 |
| in-flight duplicate | `ALREADY_EXISTS` | `IDEMPOTENCY_CHECK` | true | 3000 |
| rate limited | `RESOURCE_EXHAUSTED` | `VALIDATE_REQUEST` | true | 3000 |
| assembly retrieval failed | `UNAVAILABLE` | `ASSEMBLY_RETRIEVE` | true | 1000 |
| assembly ranking failed | `INTERNAL` | `ASSEMBLY_RANK` | true | 1000 |
| assembly budget failed | `INTERNAL` | `ASSEMBLY_BUDGET` | true | 1000 |
| model timeout | `DEADLINE_EXCEEDED` | `GENERATE_WITH_MODEL` | true | 1500 |
| model upstream unavailable | `UNAVAILABLE` | `GENERATE_WITH_MODEL` | true | 1500 |
| invalid patch shape | `INTERNAL` | `NORMALIZE_PATCH_OPS` | false | - |
| stream write failure | `UNAVAILABLE` | `EMIT_STREAM` | true | 1000 |

## 受け入れ条件（DoD）

* すべての失敗で `trace_id` が付与される
* frontend が `retryable` と `retry_after_ms` だけで再試行制御できる
* 同一条件で code/stage が変動しない
