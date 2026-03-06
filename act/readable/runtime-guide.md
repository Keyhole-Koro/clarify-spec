# Act Runtime Guide（Human View）

## 1リクエストの流れ

1. Frontend が `RunActRequest` 送信
2. `act-api` が auth/sid/csrf/authz/idempotency
3. `act-adk-worker` が context assembly
4. worker が Gemini 実行
5. worker が `patch_ops` 正規化
6. `act-api` が stream返却

## 障害時の見方

* 認証系: `AUTHN`, `AUTHZ`
* assembly系: `ASSEMBLY_RETRIEVE`, `ASSEMBLY_BUDGET`
* 生成系: `GENERATE_WITH_MODEL`
* 出力系: `NORMALIZE_PATCH_OPS`, `EMIT_STREAM`

## どこを読めばよいか

* API側の詳細:
  * `act/act-api/specs/runtime/runact-implementation.md`
* Worker側の詳細:
  * `act/act-adk-worker/specs/act-adk-runtime.md`
  * `act/act-adk-worker/specs/context/context-assembly-execution-profile.md`
* エラー最終表:
  * `act/act-api/specs/reliability/error-mapping-matrix.md`
