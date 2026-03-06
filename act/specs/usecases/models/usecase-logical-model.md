# Usecase Logical Model

ER図を実装可能な論理モデルに落とした定義。
型は実装言語非依存で、必須/一意/参照制約を中心に記述する。

## 1. Entity Definitions

### USER

| Field | Type | Required | Constraint |
|---|---|---|---|
| `uid` | string | yes | PK |
| `email` | string | yes | unique |
| `provider` | string | yes | `google.com` 固定 |
| `created_at` | timestamp | yes | - |

### WORKSPACE

| Field | Type | Required | Constraint |
|---|---|---|---|
| `workspace_id` | string | yes | PK |
| `name` | string | yes | 1..120 chars |
| `created_by` | string | yes | FK -> `USER.uid` |
| `created_at` | timestamp | yes | - |
| `status` | enum | yes | `active/archived` |

### WORKSPACE_MEMBER

| Field | Type | Required | Constraint |
|---|---|---|---|
| `workspace_id` | string | yes | FK -> `WORKSPACE.workspace_id` |
| `uid` | string | yes | FK -> `USER.uid` |
| `role` | enum | yes | `owner/member` |
| `joined_at` | timestamp | yes | - |

Composite key: `(workspace_id, uid)`

### TREE

| Field | Type | Required | Constraint |
|---|---|---|---|
| `tree_id` | string | yes | PK |
| `workspace_id` | string | yes | FK -> `WORKSPACE.workspace_id` |
| `title` | string | yes | 1..200 chars |
| `created_at` | timestamp | yes | - |

### NODE

| Field | Type | Required | Constraint |
|---|---|---|---|
| `id` | string | yes | PK |
| `tree_id` | string | yes | FK -> `TREE.tree_id` |
| `kind` | enum | yes | `ACT_ROOT/MARKDOWN/SECTION/LIST/NODE` |
| `parent_id` | string | no | FK -> `NODE.id` (same tree) |
| `title` | string | no | - |
| `content_md` | string | no | - |
| `metadata` | json | no | - |
| `updated_at` | timestamp | yes | - |

### EDGE

| Field | Type | Required | Constraint |
|---|---|---|---|
| `id` | string | yes | PK |
| `tree_id` | string | yes | FK -> `TREE.tree_id` |
| `source_id` | string | yes | FK -> `NODE.id` |
| `target_id` | string | yes | FK -> `NODE.id` |
| `edge_type` | enum | yes | `structure/context` |
| `created_at` | timestamp | yes | - |

### ACT_REQUEST

| Field | Type | Required | Constraint |
|---|---|---|---|
| `request_id` | string | yes | UUID |
| `uid` | string | yes | FK -> `USER.uid` |
| `workspace_id` | string | yes | FK -> `WORKSPACE.workspace_id` |
| `tree_id` | string | yes | FK -> `TREE.tree_id` |
| `act_type` | enum | yes | `EXPLORE/CONSULT/INVESTIGATE` |
| `created_at` | timestamp | yes | - |

Composite unique key: `(uid, workspace_id, request_id)`

### ACT_RUN

| Field | Type | Required | Constraint |
|---|---|---|---|
| `trace_id` | string | yes | PK |
| `request_id` | string | yes | FK -> `ACT_REQUEST.request_id` |
| `status` | enum | yes | `running/done/error` |
| `started_at` | timestamp | yes | - |
| `ended_at` | timestamp | no | - |

### RUN_EVENT

| Field | Type | Required | Constraint |
|---|---|---|---|
| `trace_id` | string | yes | FK -> `ACT_RUN.trace_id` |
| `seq` | int | yes | >= 1 |
| `text_delta` | string | no | - |
| `patch_ops` | json | no | `upsert/append_md` のみ |
| `terminal` | enum | no | `done/error` |

Composite key: `(trace_id, seq)`

### STREAM_PART

| Field | Type | Required | Constraint |
|---|---|---|---|
| `trace_id` | string | yes | FK -> `ACT_RUN.trace_id` |
| `seq` | int | yes | FK -> `RUN_EVENT.seq` |
| `index` | int | yes | >= 0 |
| `text` | string | yes | - |
| `thought` | bool | yes | - |

Composite key: `(trace_id, seq, index)`

### ERROR_INFO

| Field | Type | Required | Constraint |
|---|---|---|---|
| `code` | enum | yes | contract準拠 |
| `message` | string | yes | - |
| `retryable` | bool | yes | - |
| `stage` | enum | yes | `ErrorStage` 準拠 |
| `trace_id` | string | yes | FK -> `ACT_RUN.trace_id` |
| `retry_after_ms` | int | no | >= 0 |

### INVITE_TOKEN

| Field | Type | Required | Constraint |
|---|---|---|---|
| `token_id` | string | yes | PK |
| `workspace_id` | string | yes | FK -> `WORKSPACE.workspace_id` |
| `issued_by` | string | yes | FK -> `USER.uid` |
| `expires_at` | timestamp | yes | must be future at issue time |
| `consumed_by` | string | no | FK -> `USER.uid` |
| `consumed_at` | timestamp | no | - |

## 2. Referential Rules (MUST)

* `ACT_REQUEST.workspace_id == TREE.workspace_id`
* `WORKSPACE_MEMBER(workspace_id, uid)` が存在しない場合は `PERMISSION_DENIED`
* `NODE.parent_id` は同一 `tree_id` 内の `NODE.id` のみ許可
* `EDGE.source_id` / `EDGE.target_id` は同一 `tree_id` 内のノードのみ
* `RUN_EVENT.terminal=error` の場合は `ERROR_INFO` を必須で付与

## 3. Idempotency / Uniqueness

* 重複判定キー: `(uid, workspace_id, request_id)`
* 同一キー再送は `ALREADY_EXISTS`
* `trace_id` はラン単位で一意

## 4. Usecase Mapping

| Usecase | Main Entities |
|---|---|
| `UC-ASK-EMPTY-01` | `ACT_REQUEST`, `ACT_RUN`, `RUN_EVENT`, `NODE` |
| `UC-ASK-CONTEXT-01` | `ACT_REQUEST`, `NODE`, `EDGE`, `RUN_EVENT` |
| `UC-RUNACT-NODE-01` | `ACTION`, `ACT_REQUEST`, `ACT_RUN`, `EDGE` |
| `UC-THINK-STREAM-01` | `RUN_EVENT`, `STREAM_PART` |
| `UC-DEEP-FALLBACK-01` | `ACT_RUN`, `RUN_EVENT`, `ERROR_INFO` |
| `UC-WORKSPACE-CREATE-01` | `WORKSPACE`, `WORKSPACE_MEMBER` |
| `UC-WORKSPACE-INVITE-01` | `INVITE_TOKEN`, `WORKSPACE_MEMBER` |
| `UC-WORKSPACE-AUTHZ-01` | `WORKSPACE_MEMBER`, `ACT_REQUEST`, `TREE` |
