# Act Architecture

## 目的

Act の実行経路を `Interaction / Context Assembly / Knowledge Pipeline` の3層で固定し、ADK導入時の責務境界を明確化する。

## スコープ / 非スコープ

* スコープ: RunActオンライン経路、認証、ADK連携、データ参照
* 非スコープ: Organize内部処理詳細

## 前提 / 参照

* `context/model/topic-model.md`
* `context/assembly/core.md`
* `context/assembly/bundle-schema.md`
* `act/act-api/specs/runtime/adk-service-integration.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `firestore/schema.md`

## 構成図

```mermaid
flowchart LR
  U[User] --> FE[Frontend<br>Next.js + ReactFlow]
  FE -->|ID Token| AUTH[Firebase Auth]
  FE -->|RunAct stream| API[Cloud Run<br>Act API (Go)]

  API -->|verify| AUTH
  API --> ADK[Cloud Run<br>ADK Worker (Python)]
  ADK -->|read-only| FS[(Firestore topic graph)]
  ADK -->|read-only| GCS[(GCS versioned docs)]
  ADK --> VX[Vertex AI]
  VX --> ADK
  ADK --> API
  API --> FE

  ORG[Organize Pipeline] --> FS
  ORG --> GCS
```

## リクエスト経路（RunAct）

1. Frontend が ID Token を取得
2. Frontend が `RunAct(topic_id, tree_id?, request_id, ...)` 呼び出し
3. Go Act API が auth/authz と idempotency を検証
4. Go Act API が ADK Worker へ実行委譲
5. ADK Worker が Context Assembly + LLM 実行
6. Go Act API が `RunActEvent` へ変換してstream返却
7. Frontend は Draft反映し、必要時のみ Organize commit

## コンポーネント責務

* Frontend: stream反映、thought表示、commit起点
* Act API (Go): 認証認可、契約変換、stream配信
* ADK Worker: Assembly、tool orchestration、モデル実行、Act memory の管理
* Organize: 知識正本更新（write-only）
* Firestore/GCS: topic正本ストア

## MUST / 制約

* 知識正本キーは `topic_id`
* `tree_id` は UI scope のみ
* ADK Worker は Firestore/GCS に書き込まない
* Organizeは RunAct stream を emit しない
* 認証は Firebase Auth `google.com` のみ
* Act memory は実行中だけ有効な揮発状態であり、知識正本にしない
* stream中の ephemeral node、未確定候補、途中推論は Act memory に置く
* 確定して再利用したいデータは Organize 経由でのみ Firestore/GCS に昇格する
* Firestore は確定済み軽量メタと参照、GCS は大きい実体を担当する

## エラー / 例外

* Auth失敗: `UNAUTHENTICATED`
* Topicアクセス違反: `PERMISSION_DENIED`
* ADK worker到達不可: `UNAVAILABLE`
* LLM timeout: `DEADLINE_EXCEEDED`

## 完了条件（DoD）

* ADK導入後も `RunAct` 契約が不変
* 3層責務が明確に分離
* topic中心参照が文書化されている
