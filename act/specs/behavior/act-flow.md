# Act Flow 仕様（詳細版）

## 1. 目的

`RunAct` のオンライン実行フローを、実装時に判断不要な粒度で固定する。  
対象は `Frontend -> Act API -> ADK Worker -> Vertex AI -> Frontend stream` の全経路。

## 2. スコープ / 非スコープ

* スコープ:
  * 認証/認可/セッション/冪等を含む RunAct 実行順序
  * Context Assembly の read-only 境界
  * ストリーミング返却とエラー終端規約
  * Cloud Run / Redis / Firestore / GCS / Vertex AI の利用責務
* 非スコープ:
  * Organize A0-A7 の内部アルゴリズム
  * IaC/Terraform/CI の実装

## 3. 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runtime/runact-implementation.md`
* `act/act-api/specs/runtime/adk-service-integration.md`
* `act/act-api/specs/security/session-and-auth-boundary.md`
* `context/assembly/core.md`
* `context/assembly/bundle-schema.md`
* `firestore/schema.md`
* `organize/specs/pipeline/core.md`

## 4. 登場技術・クラウドサービス

### 4.1 Frontend

* Next.js (App Router)
* React / TypeScript
* React Flow
* Zustand
* Connect-Web client
* Firebase Auth JS SDK（Google Sign-In）

### 4.2 Act Backend

* Go
* Connect RPC（server-streaming）
* Firebase Admin SDK（ID Token 検証）
* Redis client（sid補助、短期ロック）
* OpenTelemetry（trace/metric）

### 4.3 ADK Worker

* Python
* Google ADK（Agent Development Kit）
* FastAPI（Act API からの内部呼び出し口）
* Vertex AI SDK（Gemini）

### 4.4 Google Cloud

* Cloud Run:
  * `act-api`（Go）
  * `act-adk-worker`（Python）
* Firebase Auth（Google provider限定）
* Firestore（workspace/topic/nodes など構造メタ）
* Cloud Storage（evidence本文、versionedオブジェクト）
* Memorystore for Redis（sid補助）
* Serverless VPC Access（Cloud Run -> Redis private接続）
* Vertex AI（Gemini 3 Flash / Web Grounding / Deep Research）
* Cloud Logging
* Cloud Monitoring
* Secret Manager（APIキー/設定の秘匿）

## 5. アーキテクチャフロー

```mermaid
flowchart LR
  U[User] --> FE[Next.js Frontend<br>React Flow + Zustand]
  FE --> AUTH[Firebase Auth]
  FE -->|RunAct stream| API[Cloud Run<br>act-api (Go + Connect)]

  API -->|verify id token| AUTH
  API --> REDIS[(Memorystore Redis)]
  API --> ACL[Access Control<br>workspace/topic check]
  API --> IDEM[Idempotency<br>uid + workspace_id + request_id]
  API --> ADK[Cloud Run<br>act-adk-worker (Python + ADK)]

  ADK --> ASM[Context Assembly<br>read-only]
  ASM --> FS[(Firestore)]
  ASM --> GCS[(Cloud Storage)]
  ADK --> VX[Vertex AI Gemini]
  VX --> ADK
  ADK --> API
  API --> FE

  ORG[Organize Pipeline] --> FS
  ORG --> GCS
```

## 6. 正常フロー（詳細）

1. Frontend が Firebase Auth で Google ログインし ID Token を取得
2. Frontend が `RunActRequest` を送信  
3. `act-api` が Connect middleware で以下を順に検証
4. Authn: ID Token 検証、`provider=google.com` を確認
5. SID: Redis で sid 補助状態を評価（`soft|strict`）
6. CSRF: Double Submit（cookie + header）照合
7. Authz: `user -> workspace membership -> topic access` 検証
8. Idempotency: `(uid, workspace_id, request_id)` で重複排除
9. `act-api` が `act-adk-worker` へ内部実行要求
10. ADK Worker が Context Assembly を実行
11. Assembly が Firestore/GCS を read-only 参照し `PromptBundle` 生成
12. ADK Worker が Vertex AI を呼び出し、本文/thinkthrough/grounding を取得
13. ADK Worker が出力を `PatchOp` 制約（`upsert` / `append_md`）へ正規化
14. `act-api` が `RunActEvent` として server-streaming 返却
15. Frontend が受信イベントを Zustand に適用しキャンバス更新
16. 必要に応じてユーザーが別経路で Organize commit を実行

## 7. 異常フロー（error/retryable/stage）

* Authn失敗: `UNAUTHENTICATED`, `retryable=false`, `stage=AUTHENTICATE`
* Authz失敗: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHORIZE`
* CSRF不一致: `PERMISSION_DENIED`, `retryable=false`, `stage=VALIDATE_REQUEST`
* Redis不達 (`soft`): 継続 + degradeログ, `stage=SID_VALIDATE`
* Redis不達 (`strict`): `UNAVAILABLE` または `UNAUTHENTICATED`
* 冪等重複: 既存結果返却または重複抑止終端, `stage=IDEMPOTENCY_CHECK`
* Assembly参照失敗: `UNAVAILABLE`, `retryable=true`, `stage=ASSEMBLY_RETRIEVE`
* Assembly予算超過: degrade継続, diagnosticsに `truncation_reason` を残す
* ADK worker不達: `UNAVAILABLE`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* Vertexタイムアウト: `DEADLINE_EXCEEDED`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* 正規化不正: `INTERNAL`, `retryable=false`, `stage=NORMALIZE_PATCH_OPS`
* stream途中障害: `error` 終端（`done` なし）

## 8. 数値パラメータ（運用デフォルト）

* `act-api` Cloud Run:
  * `concurrency=40`
  * `timeout=120s`
  * `memory=2Gi`
  * `min-instances=0`
  * `max-instances=20`
* `act-adk-worker` Cloud Run:
  * `concurrency=10`
  * `timeout=120s`
  * `memory=2Gi`
  * `min-instances=0`
  * `max-instances=20`
* Session/Redis:
  * `SID_TTL_SECONDS=86400`
  * `SID_REQ_TTL_SECONDS=900`
  * `SID_LOCK_TTL_SECONDS=10`
  * `SID_ENFORCE_MODE=soft`（当日デフォルト）
* 追加詳細は `act/specs/quality/backend-parameter-index.md` を正本とする

## 9. 受け入れ条件（DoD）

* 実装者がサービス境界を誤らない
* `act-api` と `act-adk-worker` の責務が混在しない
* Assembly の read-only 制約が明示されている
* Redis が認証正本でないことが明示されている
* 主要失敗ケースに `code/retryable/stage` がある
* Frontend が stream適用のみで状態更新できる

## 10. 実装メモ（最小）

* オンライン経路で Firestore/GCS write を行わない
* `topic_id` を知識正本キーとして扱う
* `tree_id` は UI スコープ用途のみ
* `trace_id` を Frontend -> API -> ADK -> Vertex で貫通させる
