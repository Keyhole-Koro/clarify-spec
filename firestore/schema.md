# Firestore スキーマ仕様（Topic-Centric 統合版）

目的: Topic中心モデルで、Act/Organize/Frontendが同じ正本を参照できるようにする。

## 1. スコープ

* 認証/認可に必要な workspace 境界
* topic/draft/outline/node/evidence の知識正本
* act run と event ledger/lease/version 管理

## 2. 論理ER（Core）

```mermaid
erDiagram
  USER ||--o{ WORKSPACE_MEMBER : belongs_to
  WORKSPACE ||--o{ WORKSPACE_MEMBER : has
  WORKSPACE ||--o{ TOPIC : has
  TOPIC ||--o{ DRAFT : has
  TOPIC ||--o{ OUTLINE : has
  TOPIC ||--o{ NODE : has
  TOPIC ||--o{ EDGE : has
  NODE ||--o{ EVIDENCE : cites
  TOPIC ||--o{ ACT_RUN : has
  TOPIC ||--o{ INPUT : has
  TOPIC ||--o{ INPUT_PROGRESS : has
  TOPIC ||--o{ ATOM : has
  TOPIC ||--o{ PIPELINE_BUNDLE : has
  NODE ||--o{ INDEX_ITEM : indexed_by

  WORKSPACE {
    string workspace_id PK
    string name
    string created_by
    timestamp created_at
    string status
  }

  TOPIC {
    string topic_id PK
    string workspace_id FK
    string title
    string status
    int schema_version
    int latest_draft_version
    int latest_outline_version
    timestamp updated_at
  }

  TOPIC ||--o{ TOPIC_SCHEMA : has

  TOPIC_SCHEMA {
    string topic_id PK
    int version PK
    string status
    string[] node_kinds
    string[] relation_types
    map attribute_defs
    map index_feature_defs
    timestamp created_at
  }

  DRAFT {
    string topic_id PK
    int version PK
    ref summary_md_ref
    string[] source_atom_ids
    timestamp created_at
  }

  OUTLINE {
    string topic_id PK
    int version PK
    ref summary_md_ref
    ref map_md_ref
    timestamp published_at
  }

  NODE {
    string node_id PK
    string topic_id FK
    string kind
    string title
    string parent_id
    ref context_summary_ref
    timestamp updated_at
  }

  EDGE {
    string edge_id PK
    string topic_id FK
    string source_id FK
    string target_id FK
    string relation
    number order_key
  }

  EVIDENCE {
    string evidence_id PK
    string node_id FK
    string source_type
    string url
    string gcs_uri
    string generation
    string sha256
    float confidence
  }

  INPUT {
    string input_id PK
    string topic_id FK
    string status "received | stored | extracted | error"
    string content_type
    ref raw_ref
    ref extracted_ref
    timestamp created_at
    timestamp updated_at
  }

  INPUT_PROGRESS {
    string input_id PK
    string topic_id FK
    string workspace_id FK
    string status "uploaded | extracting | atomizing | resolving_topic | updating_draft | completed | failed"
    string current_phase
    string last_event_type
    string trace_id
    string resolution_mode
    string resolved_topic_id
    string error_code
    string error_message
    timestamp phase_updated_at
    timestamp completed_at
    timestamp updated_at
  }

  ATOM {
    string atom_id PK "sha256(topicId + inputId + claimIndex)"
    string topic_id FK
    string source_input_id FK
    int claim_index
    string title
    string claim
    string kind "fact | definition | relation | opinion | temporal"
    float confidence
    int reissue_count "default 0"
    timestamp created_at
  }

  PIPELINE_BUNDLE {
    string bundle_id PK
    string topic_id FK
    int source_draft_version
    int schema_version
    int atom_count
    timestamp created_at
    timestamp applied_at "null until A3 applies"
  }

  INDEX_ITEM {
    string index_item_id PK
    string topic_id FK
    string node_id FK
    int schema_version
    int outline_version
    float relation_importance
    float recency
    float confidence
    int evidence_count
    int edge_count
    int depth
    timestamp updated_at
  }

  ACT_RUN {
    string run_id PK
    string topic_id FK
    string request_id
    string uid
    string status
    timestamp started_at
    timestamp ended_at
  }
```

## 3. 論理ER（Operations）

```mermaid
erDiagram
  WORKSPACE ||--o{ EVENT_LEDGER : has
  WORKSPACE ||--o{ LEASE : has
  WORKSPACE ||--o{ VERSION_CURSOR : has

  EVENT_LEDGER {
    string ledger_id PK
    string idempotency_key_hash
    string topic_id
    string agent
    string status
    timestamp started_at
    timestamp finished_at
  }

  LEASE {
    string resource_key PK
    string topic_id
    string owner
    timestamp expires_at
  }

  VERSION_CURSOR {
    string resource_key PK
    string topic_id
    int version
    timestamp updated_at
  }
```

## 4. 物理パス（推奨）

* `workspaces/{workspaceId}`
* `workspaces/{workspaceId}/members/{uid}`
* `workspaces/{workspaceId}/invites/{inviteId}`
* `workspaces/{workspaceId}/topics/{topicId}`
* `workspaces/{workspaceId}/topics/{topicId}/drafts/{version}`
* `workspaces/{workspaceId}/topics/{topicId}/outlines/{version}`
* `workspaces/{workspaceId}/topics/{topicId}/schemas/{version}`
* `workspaces/{workspaceId}/topics/{topicId}/nodes/{nodeId}`
* `workspaces/{workspaceId}/topics/{topicId}/edges/{edgeId}`
* `workspaces/{workspaceId}/topics/{topicId}/nodes/{nodeId}/evidence/{evidenceId}`
* `workspaces/{workspaceId}/topics/{topicId}/inputs/{inputId}`
* `workspaces/{workspaceId}/topics/{topicId}/inputProgress/{inputId}`
* `workspaces/{workspaceId}/topics/{topicId}/atoms/{atomId}`
* `workspaces/{workspaceId}/topics/{topicId}/pipelineBundles/{bundleId}`
* `workspaces/{workspaceId}/topics/{topicId}/indexItems/{indexItemId}`
* `workspaces/{workspaceId}/topics/{topicId}/actRuns/{runId}`
* `workspaces/{workspaceId}/topics/{topicId}/actRuns/{runId}/events/{seq}`
* `workspaces/{workspaceId}/topics/{topicId}/organizeOps/{opId}`
* `workspaces/{workspaceId}/eventLedger/{hash}`
* `workspaces/{workspaceId}/leases/{resourceKey}`
* `workspaces/{workspaceId}/versions/{resourceKey}`

## 5. 主要制約（MUST）

* `topic.workspace_id == request.workspace_id`
* `members/{uid}` が無ければ `PERMISSION_DENIED`
* `ACT_RUN` は `(uid, request_id)` で topic内一意
* `EDGE.source/target` は同一 topic 内に限定
* `schema_version` は topic 内で単調増加し、`topics/{topicId}/schemas/{version}` と一致する
* `latest_draft_version`, `latest_outline_version` は単調増加
* 本文はGCS versioned ref（`gcsUri/generation/sha256`）を保持
* Firestore は確定済みメタ/関係/権限境界/検索キーの正本とし、stream中の高頻度一時状態は保持しない
* ただし upload 処理進捗は UI 露出に必要なため `inputProgress/{inputId}` を正本として保持してよい
* `actRuns/events` は監査と短期再送補助のための記録に限定し、Act memory の代替にしない
* topic ごとに進化できるのは knowledge schema のみとし、workspace/authz/path/id など platform schema は topic ごとに変化させない
* node/edge/index は適用に使った `schema_version` を参照可能にする

## 6.1 Upload Progress 正本

フロントの upload 進捗表示は Pub/Sub を直接参照せず、Firestore の `inputProgress/{inputId}` を正本とする。

MUST:

* 各 phase 完了時に `inputProgress/{inputId}` を単調更新する
* retry 中は status を巻き戻さない
* 長時間処理だけを理由に `failed` を設定しない
* `failed` は backend が永続失敗を明示するときのみ設定する
* `completed` では `resolvedTopicId` と `resolutionMode` を保持できるようにする

推奨フィールド:

* `status`
* `currentPhase`
* `lastEventType`
* `traceId`
* `resolvedTopicId`
* `resolutionMode`
* `phaseUpdatedAt`
* `completedAt`
* `errorCode`
* `errorMessage`

## 6. トランザクション境界

* Topic参加/招待: invite消費 + member追加を同一transaction
* Draft更新: `latest_draft_version` CAS
* Topic schema 更新: `schema_version` CAS
* Outline更新: `latest_outline_version` CAS
* Bundle適用: `appliedAt` CAS
* Lease取得: `leases/{resourceKey}` compare-and-set

## 7. 監査/保持

* `actRuns/events`: TTLで短期保持
* `eventLedger`: 冪等期間に合わせて保持
* error系は `trace_id` でログ相互参照

## 8. ノート

* `tree_id` はUI表示境界として利用可
* 知識正本の主キーは `topic_id`
* 長文本文、raw observation、生成物、snapshot 実体は Firestore 本体ではなく GCS に置く
* `topic_schema` は topic ごとの知識構造を表す。例: node kind, relation type, attribute set, index feature
