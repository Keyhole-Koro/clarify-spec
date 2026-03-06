# Connect サーバ公開仕様

## 目的

`ActService.RunAct` を Connect RPC として公開し、フロント実装がそのまま接続できる運用境界を定義する。

## 公開対象

* Service: `ActService`
* RPC: `RunAct`（server-streaming）
* 実装スタック: Go + connect-go

## 前提仕様

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runtime/runact-implementation.md`
* `act/act-api/specs/security/access-control-middleware.md`
* `act/act-api/specs/security/session-and-auth-boundary.md`
* `act/act-api/specs/platform/cloudrun-redis-topology.md`
* `act/act-api/specs/security/cookie-cors-policy.md`
* `act/act-api/specs/security/internal-service-auth.md`

## サーバ要件（MUST）

* streaming応答を途中切断なく返せる
* deadline/timeout超過時に明示エラーを返す
* requestごとに traceId を付与
* `request_id`（UUID）を受け取り、冪等キー重複を判定できる
* sid は HttpOnly Cookie を正本として扱う
* state-changing endpoint では CSRF Double Submit を検証する
* workspace/uid の認可チェックを行う
* Vertex AI 認証情報を解決し、Gemini呼び出し可能であること
* `thinking_config` / `research_config` を解釈できること

## 環境境界

* `RPC_BASE_URL` でendpoint切替
* dev/prod で公開ポリシーを分離
* CORSはfrontend originを明示許可
* `VERTEX_PROJECT_ID` / `VERTEX_LOCATION` を環境ごとに固定
* `GOOGLE_APPLICATION_CREDENTIALS` もしくは Workload Identity を使用
* `REDIS_HOST` / `REDIS_PORT` / `REDIS_DB` を設定
* `SID_ENFORCE_MODE` は当日 `soft` を既定とする

## セキュリティ

* 認証情報（uid/workspace）を必ず検証
* Firebase ID Token の `firebase.sign_in_provider` は `google.com` のみ許可
* middleware で `uid一致 -> workspace所属 -> tree所属` を共通検証する
* workspace越境は `PERMISSION_DENIED` で拒否
* tree不存在は `FAILED_PRECONDITION` で返す
* ログへ機微情報本文を直接出さない
* sid は認証正本にせず、Firebase tokenを常に優先する

## 障害時挙動

* upstream障害時は `UNAVAILABLE` / `DEADLINE_EXCEEDED`
* 内部例外は `INTERNAL`
* `ErrorInfo.stage` と `ErrorInfo.retryable` を付与して返す
* Redis不達時は機能別切替（RunAct継続、sid依存機能はdegrade）
* 失敗ログに `traceId` を必須出力

## 完了条件（DoD）

* フロントから `RunAct` stream接続が成立
* 認可失敗時の拒否が確認できる
* traceIdでサーバログ追跡が可能
* Goサーバで stream 中断時のリソースリークがない
