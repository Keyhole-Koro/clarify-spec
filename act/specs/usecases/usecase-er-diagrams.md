# Usecase ER Diagrams

Usecaseごとのデータ関係をER図で示す。
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
