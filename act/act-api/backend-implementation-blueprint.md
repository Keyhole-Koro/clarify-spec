# Act Backend Implementation Blueprint

## 目的

ハッカソン当日に `act/act-api` と `act/act-adk-worker` を迷わず実装できるよう、ライブラリ選定、ディレクトリ構成、Cloud Run構成、Redisの責務を固定する。

## スコープ / 非スコープ

* スコープ: Go API + ADK Worker の実装方針、ランタイム構成、依存ライブラリ、Redis利用目的
* 非スコープ: 実コード、IaCコード、CI定義

## 前提 / 参照

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runact-implementation.md`
* `act/act-api/specs/adk-service-integration.md`
* `act/act-api/specs/cloudrun-redis-topology.md`
* `act/specs/quality/backend-parameter-index.md`

## アーキテクチャ要約

* `act-api`（Go / Cloud Run）:
  * Connect RPC公開
  * Firebase Auth検証
  * workspace/topic認可
  * request_id冪等判定
  * ADK Worker呼び出し
  * RunAct stream返却
* `act-adk-worker`（Python / Cloud Run）:
  * Context Assembly（read-only）
  * Vertex AI呼び出し
  * 思考/本文/groundingを内部イベント化
  * `PatchOp` 制約に正規化

## 推奨ライブラリ（固定）

### Go (`act-api`)

* HTTP/Connect:
  * `connectrpc.com/connect`
  * `connectrpc.com/grpchealth`
* Auth / GCP:
  * `firebase.google.com/go/v4`
  * `google.golang.org/api/idtoken`（必要時）
* Data:
  * `cloud.google.com/go/firestore`
  * `cloud.google.com/go/storage`
* Redis:
  * `github.com/redis/go-redis/v9`
* Logging/metrics:
  * `go.opentelemetry.io/otel`
  * `go.opentelemetry.io/otel/sdk/metric`
* Config:
  * 標準 `os` + 明示的env読み込み（隠れ設定を作らない）

### Python (`act-adk-worker`)

* ADK/LLM:
  * Google Agent Development Kit（ADK）
  * Vertex AI SDK（Gemini呼び出し）
* API:
  * `fastapi`
  * `uvicorn`
* Data access (read-only):
  * `google-cloud-firestore`
  * `google-cloud-storage`

## ディレクトリ構成（提案）

```text
act/
  README.md
  act-api/
    README.md
    backend-implementation-blueprint.md
    cmd/
      act-api/
        main.go
    internal/
      app/                 # DI, startup
      transport/connect/   # Connect handlers
      auth/                # Firebase token verify
      access/              # workspace/topic access control
      session/             # sid/csrf/redis policy
      runact/              # runact orchestration
      adkclient/           # call act-adk-worker
      stream/              # RunActEvent mapping
      store/               # firestore/gcs repositories (read)
      idempotency/         # request_id dedupe
      observability/       # logs/metrics/tracing
      config/              # env parsing
    test/
      integration/
      e2e/
  act-adk-worker/
    README.md
    app/
      main.py
      routes/
      workflow/            # ADK workflow nodes
      assembly/            # context assembly stages
      vertex/              # model adapters
      normalize/           # PatchOp normalization
      store/               # firestore/gcs read adapters
      observability/
      config/
    tests/
```

## Cloud Run 構成

### `act-api`（Go）

* `concurrency=40`
* `timeout=120s`
* `memory=2Gi`
* `min-instances=0`
* `max-instances=20`
* 認証: Firebase ID Token（`google.com` provider必須）
* VPC Connector経由でRedisへ接続

### `act-adk-worker`（Python）

* `concurrency=10`
* `timeout=120s`
* `memory=2Gi`
* `min-instances=0`
* `max-instances=20`
* 役割は read-only（Firestore/GCS write禁止）

## Redis で何をするか（責務）

Redisは認証の正本ではなく、実行安定化の補助に限定する。

* `sid` セッション補助状態の保持（TTL管理）
* request単位の短期ロック（重複同時実行の緩和）
* 軽量レートゲート
* last_seen 更新

やらないこと:

* ユーザー認証の正本化
* workspace権限判定の正本化
* 永続データ保存

## リクエスト処理順（MUST）

1. Authn（Firebase token verify）
2. SID（Redis、`soft|strict`）
3. CSRF検証
4. AccessControl（`user -> membership -> topic`）
5. Idempotency（`uid + workspace_id + request_id`）
6. ADK Worker実行
7. Stream返却

## 障害時挙動（要点）

* Redis不達 + `SID_ENFORCE_MODE=soft`: RunAct継続（degradeログ）
* Redis不達 + `strict`: `UNAUTHENTICATED` または `UNAVAILABLE`
* ADK worker不達: `UNAVAILABLE`（retryable）
* ADK worker timeout: `DEADLINE_EXCEEDED`（retryable）
* 正規化失敗: `INTERNAL`（non-retryable）

## 受け入れ条件（DoD）

* ライブラリ選定がGo/Pythonで固定されている
* 実装ディレクトリ構成の迷いがない
* Cloud Runのデフォルト値が仕様に一致
* Redisの責務が「補助」に限定されている
