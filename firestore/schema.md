# Firestore スキーマ仕様（統合版）

目的: DB周りの論理モデルとFirestore物理配置を、実装に使える粒度で固定する。

## 1. スコープ

* Act実行に必要な認証/認可/冪等
* 知識グラフ（tree/node/edge/evidence）
* Organize運用（ledger/lease/version）

## 2. 論理ER（Core Knowledge Graph）

```mermaid
erDiagram
  USER ||--o{ WORKSPACE_MEMBER : belongs_to
  WORKSPACE ||--o{ WORKSPACE_MEMBER : has
  WORKSPACE ||--o{ TREE : has
  TREE ||--o{ NODE : has
  TREE ||--o{ EDGE : has
  NODE ||--o{ EVIDENCE : cites

  WORKSPACE {
    string workspace_id PK
    string name
    string created_by
    timestamp created_at
    string status
  }

  WORKSPACE_MEMBER {
    string workspace_id PK
    string uid PK
    string role
    timestamp joined_at
  }

  TREE {
    string tree_id PK
    string workspace_id FK
    string title
    string[] entry_node_ids "optional"
    timestamp created_at
    timestamp updated_at
  }

  NODE {
    string node_id PK
    string tree_id FK
    string kind
    string title
    string content_md
    string parent_id
    boolean layout_locked
    json layout
    timestamp updated_at
  }

  EDGE {
    string edge_id PK
    string tree_id FK
    string source_id FK
    string target_id FK
    string edge_type
    number order_key
    timestamp created_at
  }

  EVIDENCE {
    string evidence_id PK
    string node_id FK
    string type
    string url
    string ref
    string excerpt
    timestamp created_at
  }
```

## 3. 論理ER（Act Runtime / Idempotency）

```mermaid
erDiagram
  USER ||--o{ ACT_REQUEST : sends
  WORKSPACE ||--o{ ACT_REQUEST : receives
  TREE ||--o{ ACT_REQUEST : targets
  ACT_REQUEST ||--o{ ACT_RUN : spawns
  ACT_RUN ||--o{ RUN_EVENT : emits
  RUN_EVENT ||--o{ STREAM_PART : has
  ACT_RUN ||--o{ ERROR_INFO : may_end_with

  ACT_REQUEST {
    string uid PK
    string workspace_id PK
    string request_id PK
    string tree_id
    string act_type
    timestamp created_at
  }

  ACT_RUN {
    string trace_id PK
    string request_id
    string status
    timestamp started_at
    timestamp ended_at
  }

  RUN_EVENT {
    string trace_id PK
    int seq PK
    string text_delta
    json patch_ops
    string terminal
  }

  STREAM_PART {
    string trace_id PK
    int seq PK
    int idx PK
    string text
    boolean thought
  }

  ERROR_INFO {
    string trace_id PK
    string code
    string message
    boolean retryable
    string stage
    int retry_after_ms
  }
```

## 4. 論理ER（Operations / Conflict Control）

```mermaid
erDiagram
  WORKSPACE ||--o{ EVENT_LEDGER : has
  WORKSPACE ||--o{ LEASE : has
  WORKSPACE ||--o{ VERSION_CURSOR : has

  EVENT_LEDGER {
    string ledger_id PK
    string idempotency_key_hash
    string state
    timestamp started_at
    timestamp finished_at
    string error_code
  }

  LEASE {
    string resource_key PK
    string holder
    timestamp acquired_at
    timestamp expires_at
  }

  VERSION_CURSOR {
    string resource_key PK
    int version
    timestamp updated_at
  }
```

## 5. Firestore 物理パス（推奨）

```txt
workspaces/{workspaceId}
workspaces/{workspaceId}/members/{uid}
workspaces/{workspaceId}/invites/{inviteId}
workspaces/{workspaceId}/trees/{treeId}
workspaces/{workspaceId}/trees/{treeId}/nodes/{nodeId}
workspaces/{workspaceId}/trees/{treeId}/edges/{edgeId}
workspaces/{workspaceId}/trees/{treeId}/nodes/{nodeId}/evidence/{evidenceId}
workspaces/{workspaceId}/actRequests/{uid_requestId}
workspaces/{workspaceId}/actRuns/{traceId}
workspaces/{workspaceId}/actRuns/{traceId}/events/{seq}
workspaces/{workspaceId}/eventLedger/{hash}
workspaces/{workspaceId}/leases/{resourceKey}
workspaces/{workspaceId}/versions/{resourceKey}
```

互換パス（旧）:

```txt
users/{uid}/trees/{treeId}
users/{uid}/trees/{treeId}/nodes/{nodeId}
users/{uid}/trees/{treeId}/edges/{edgeId}
users/{uid}/trees/{treeId}/nodes/{nodeId}/evidence/{evidenceId}
```

## 6. 主要制約（MUST）

* `request.workspace_id == tree.workspace_id`
* `members/{uid}` が無ければ `PERMISSION_DENIED`
* `ACT_REQUEST (uid, workspace_id, request_id)` は一意
* `EDGE` の `source_id/target_id` は同一 `tree_id` 内に限定
* `RUN_EVENT.terminal=error` のとき `ERROR_INFO` 必須
* `layout` は `layout_locked=true` のノードのみ保存

## 7. 推奨インデックス

* `trees`: `workspace_id + updated_at desc`
* `nodes`: `tree_id + updated_at desc`
* `edges`: `tree_id + source_id`
* `actRequests`: `uid + created_at desc`
* `actRuns`: `status + started_at desc`
* `eventLedger`: `idempotency_key_hash`（unique相当）

## 8. トランザクション境界

* Workspace参加: `invites` 消費 + `members` 追加を同一トランザクション
* Act開始: `actRequests` 作成（重複チェック）を先に確定
* ApplyPatch: `nodes/edges/evidence` を一括バッチ（CAS版数確認）
* Lease取得: `leases/{resourceKey}` を compare-and-set

## 9. レイアウト保存フロー

```mermaid
sequenceDiagram
  autonumber
  participant F as Frontend (ReactFlow)
  participant O as OrganizeService (Connect RPC)
  participant FS as Firestore

  F->>O: GetTree(treeId) + ListNodes/ListEdges
  O->>FS: read trees/nodes/edges
  FS-->>O: docs
  O-->>F: nodes + edges

  F-->>F: ELKレイアウト計算<br>(layoutLocked=trueは固定)

  Note over F: ユーザーがノードAをドラッグしてlock
  F->>O: UpdateNodeFields(A, layoutLocked=true, layout={x,y})
  O->>FS: update nodes/{A}
  FS-->>O: ok
  O-->>F: Node(updated)
```

## 10. 監査/保持

* `actRuns/events`: TTLで短期保持（デバッグ用途）
* `eventLedger`: 冪等性期間に合わせて保持
* `ERROR_INFO`: `trace_id` でログと相互参照
