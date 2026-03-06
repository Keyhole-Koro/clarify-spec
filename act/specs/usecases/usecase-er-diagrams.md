# Usecase ER Diagrams (Authorization & Scope)

## 目的

Act における「認可境界」と「実行スコープ」の関係を可視化し、実装時の検証ロジック漏れを防ぐ。

## 1. Authorization & Scope Boundary

Workspace を中心とした認可とデータの所属関係。

```mermaid
erDiagram
    USER ||--o{ WORKSPACE_MEMBER : "joins"
    WORKSPACE ||--o{ WORKSPACE_MEMBER : "has"
    
    WORKSPACE ||--o{ TOPIC : "owns (Knowledge Boundary)"
    WORKSPACE ||--o{ TREE : "owns (UI Boundary)"
    
    ACT_REQUEST }o--|| USER : "initiated by"
    ACT_REQUEST }o--|| WORKSPACE : "executed in"
    
    ACT_REQUEST }o--|| TOPIC : "targets (MUST)"
    ACT_REQUEST }o--o| TREE : "refers (OPTIONAL)"

    %% Constraints
    ACT_REQUEST ||--|| AUTH_CHECK : "validates"
```

## 2. Validation Logic Flow

Act Request 受信時の検証ロジック（Middleware / Handler）。

```mermaid
flowchart TD
    Start([RunAct Request]) --> AuthN{Valid Token?}
    AuthN -- No --> ErrAuthN[UNAUTHENTICATED]
    AuthN -- Yes --> AuthZ_Member{User is Member<br/>of Workspace?}
    
    AuthZ_Member -- No --> ErrAuthZ[PERMISSION_DENIED]
    AuthZ_Member -- Yes --> CheckTopic{Topic belongs<br/>to Workspace?}
    
    CheckTopic -- No --> ErrTopic[PERMISSION_DENIED<br/>(Cross-boundary access)]
    CheckTopic -- Yes --> HasTree{Tree ID provided?}
    
    HasTree -- No --> Exec[Execute Act]
    HasTree -- Yes --> CheckTree{Tree belongs<br/>to Workspace?}
    
    CheckTree -- No --> ErrTree[PERMISSION_DENIED<br/>(Cross-boundary access)]
    CheckTree -- Yes --> Exec
```

## 3. Usecase Specific Diagrams

詳細挙動は各usecase本文を正本とする。

## UC-ASK-EMPTY-01

```mermaid
erDiagram
  USER ||--o{ ACT_RUN : executes
  WORKSPACE ||--o{ TREE : contains
  TREE ||--o{ NODE : contains
  ACT_RUN }o--|| WORKSPACE : runs_in
  ACT_RUN }o--|| TREE : targets
  ACT_RUN ||--o{ RUN_EVENT : emits
  RUN_EVENT }o--o{ NODE : upserts_or_appends
```

## UC-ASK-CONTEXT-01

```mermaid
erDiagram
  ACT_RUN }o--o{ NODE : context_nodes
  ACT_RUN ||--o{ RUN_EVENT : emits
  RUN_EVENT }o--|| NODE : creates_act_root
  NODE ||--o{ EDGE : source
  EDGE }o--|| NODE : target
  EDGE }o--|| ACT_RUN : context_edge_of
```

## UC-RUNACT-NODE-01

```mermaid
erDiagram
  NODE ||--o{ ACTION : has
  ACTION ||--|| ACT_RUN : triggers
  ACT_RUN }o--|| NODE : anchor_node
  ACT_RUN ||--o{ RUN_EVENT : emits
  RUN_EVENT }o--|| NODE : creates_derived_root
```

## UC-THINK-STREAM-01

```mermaid
erDiagram
  ACT_RUN ||--o{ RUN_EVENT : emits
  RUN_EVENT ||--o{ STREAM_PART : has
  STREAM_PART }o--|| ACT_RUN : belongs_to
  STREAM_PART {
    boolean thought
    string text
  }
```

## UC-DEEP-FALLBACK-01

```mermaid
erDiagram
  ACT_RUN ||--o{ MODEL_ATTEMPT : has
  MODEL_ATTEMPT }o--|| LLM_PROFILE : uses
  MODEL_ATTEMPT ||--o{ RUN_EVENT : emits
  ACT_RUN ||--o{ WARNING : records
  LLM_PROFILE {
    string profile_name
  }
```

## UC-WORKSPACE-CREATE-01

```mermaid
erDiagram
  USER ||--o{ WORKSPACE : creates
  WORKSPACE ||--o{ WORKSPACE_MEMBER : has
  USER ||--o{ WORKSPACE_MEMBER : joins
  WORKSPACE ||--o{ TREE : owns
```

## UC-WORKSPACE-INVITE-01

```mermaid
erDiagram
  WORKSPACE ||--o{ INVITE_TOKEN : issues
  INVITE_TOKEN }o--|| USER : consumed_by
  USER ||--o{ WORKSPACE_MEMBER : becomes
  WORKSPACE ||--o{ WORKSPACE_MEMBER : has
```

## UC-WORKSPACE-AUTHZ-01

```mermaid
erDiagram
  USER ||--o{ WORKSPACE_MEMBER : membership
  WORKSPACE ||--o{ TREE : contains
  ACT_REQUEST }o--|| USER : from
  ACT_REQUEST }o--|| WORKSPACE : claims
  ACT_REQUEST }o--|| TREE : targets
  WORKSPACE_MEMBER }o--|| WORKSPACE : grants_access
```

## 共通（認証・認可チェック）

```mermaid
erDiagram
  USER ||--o{ AUTH_TOKEN : owns
  USER ||--o{ WORKSPACE_MEMBER : member_of
  WORKSPACE ||--o{ WORKSPACE_MEMBER : has
  WORKSPACE ||--o{ TREE : has
  ACT_REQUEST }o--|| AUTH_TOKEN : presents
  ACT_REQUEST }o--|| WORKSPACE : workspace_id
  ACT_REQUEST }o--|| TREE : tree_id
```

## データ構造（主要フィールド）

### Core Entities

| Entity | Key Fields | Notes |
|---|---|---|
| `USER` | `uid`, `email`, `provider` | `provider=google.com` を前提 |
| `WORKSPACE` | `workspace_id`, `name`, `created_by`, `created_at` | ユーザーは複数作成可 |
| `WORKSPACE_MEMBER` | `workspace_id`, `uid`, `role`, `joined_at` | `role`: `owner/member` |
| `TREE` | `tree_id`, `workspace_id`, `title` | 認可で `workspace_id` 整合必須 |
| `NODE` | `id`, `tree_id`, `kind`, `parent_id`, `content_md` | `kind` は `ACT_ROOT` など |
| `EDGE` | `id`, `source_id`, `target_id`, `edge_type` | `edge_type`: `structure/context` |

### Act Stream Entities

| Entity | Key Fields | Notes |
|---|---|---|
| `ACT_REQUEST` | `request_id`, `uid`, `workspace_id`, `tree_id`, `act_type` | 冪等キー: `(uid, workspace_id, request_id)` |
| `ACT_RUN` | `trace_id`, `request_id`, `status`, `started_at` | status: `running/done/error` |
| `RUN_EVENT` | `trace_id`, `seq`, `patch_ops[]`, `text_delta`, `terminal` | `done/error` 排他 |
| `STREAM_PART` | `text`, `thought` | thinkthrough分離表示用 |
| `ERROR_INFO` | `code`, `message`, `retryable`, `stage`, `trace_id` | `retry_after_ms` は任意 |

### Workspace Invite Entities

| Entity | Key Fields | Notes |
|---|---|---|
| `INVITE_TOKEN` | `token_id`, `workspace_id`, `issued_by`, `expires_at`, `consumed_by` | 短寿命トークン前提 |

## 構造ルール（MUST）

* `ACT_REQUEST.workspace_id` と `TREE.workspace_id` が一致しない場合は拒否
* `WORKSPACE_MEMBER` が存在しない `uid` は `PERMISSION_DENIED`
* `ACT_REQUEST.request_id` は UUID で、同一キー重複は `ALREADY_EXISTS`
* `RUN_EVENT.error` には `stage/retryable/trace_id` を必ず含める
