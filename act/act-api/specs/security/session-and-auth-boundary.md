# Session and Auth Boundary 仕様

## 目的

Firebase Auth（認証正本）と Redis sid（補助セッション）の責務境界を固定し、当日実装での判断をなくす。

## スコープ / 非スコープ

* スコープ: `RunAct` 呼び出し時の認証・セッション・CSRF検証
* 非スコープ: OAuth実装詳細、Firebaseプロジェクト設定手順

## 前提 / 参照

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runtime/connect-server.md`
* `act/act-api/specs/security/access-control-middleware.md`
* `act/act-api/specs/security/cookie-cors-policy.md`
* `act/specs/quality/backend-parameter-index.md`

## 契約（I/O）

入力:

* `Authorization: Bearer <Firebase ID Token>`（必須）
* Cookie `sid`（HttpOnly, Secure, SameSite=Lax, HTTPS only）
* Cookie `csrf_token`（JS読取可）
* Header `X-CSRF-Token`（state-changing時のみ）
* `RunActRequest.request_id`（必須）

出力:

* `RunActEvent` stream
* 失敗時 `ErrorInfo { code, retryable, stage, trace_id, retry_after_ms? }`

## 処理フロー（正常）

1. Firebase token を検証し `uid` を確定
2. request `uid` が存在する場合のみ claim `uid` と照合（互換モード）
3. `sid` Cookie を Redis で検証（softモードでは欠落許容）
4. CSRF（Double Submit）を照合
5. access-control middleware で `workspace/tree` 認可
6. handler へ委譲して `RunAct` 実行

## 異常フロー（エラーコード・retryable）

* token不正 / provider不一致: `UNAUTHENTICATED`, `retryable=false`, `stage=AUTHN`
* request uid 不一致: `UNAUTHENTICATED`, `retryable=false`, `stage=AUTHN`
* sid不正（strict）: `UNAUTHENTICATED`, `retryable=true`, `stage=SID_VALIDATE`
* csrf不一致: `PERMISSION_DENIED`, `retryable=false`, `stage=CSRF_VALIDATE`
* workspace未所属 / tree越境: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* Redis不達（soft）: 処理継続 + 警告ログ（degrade）

## 数値パラメータ

* `SID_TTL_SECONDS=86400`
* `CSRF_TTL_SECONDS=86400`
* `SID_ENFORCE_MODE=soft`（当日開始）

## 受け入れ条件（DoD）

* 認証正本が Firebase token であることを全仕様が共有
* sid が認証代替ではなく補助セッションであることが明記されている
* CSRF手順（Double Submit）が明記されている

## 実装メモ（最小）

* sid は JS で読み書きしない（HttpOnly）
* フロントは `credentials: include` と `Authorization` を併用する
