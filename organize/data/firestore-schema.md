# RPC/Firestore スキーマ仕様（edge方式）

* `users/{uid}/trees/{treeId}`
* `users/{uid}/trees/{treeId}/nodes/{nodeId}`
* `users/{uid}/trees/{treeId}/edges/{edgeId}`
* `users/{uid}/trees/{treeId}/nodes/{nodeId}/evidence/{evidenceId}`

### trees/{treeId}

* `title: string`
* `entryNodeIds: string[]`（任意：入口を固定したいなら。無ければ「親がいないnodeがroot扱い」でもOK）
* `createdAt`, `updatedAt`

### nodes/{nodeId}

* `title: string`
* `contentMd: string`
* `layoutLocked: boolean`（true のノードだけ手動配置）
* `layout?: {x:number,y:number}`（**layoutLocked=true の時だけ保存**）
* `createdAt`, `updatedAt`

### edges/{edgeId}

* `parentId: string`
* `childId: string`
* `orderKey: number`（同一parent配下の並び。ギャップ運用）
* `createdAt`, `updatedAt`

> ✅ `tags` / `relatedNodeIds` は削除
> ✅ `parentId` は node から消して **edgeで複数親を表現**
> ✅ `childrenIds` は保持しない（必要なら edge をクエリして導出）

---

## Mermaid：ER（更新版）

```mermaid
erDiagram
  USER ||--o{ TREE : owns
  TREE ||--o{ NODE : has
  TREE ||--o{ EDGE : has
  NODE ||--o{ EVIDENCE : cites

  TREE {
    string treeId PK
    string title
    string[] entryNodeIds "optional"
    timestamp createdAt
    timestamp updatedAt
  }

  NODE {
    string nodeId PK
    string title
    string contentMd
    boolean layoutLocked
    json layout "{x,y} optional"
    timestamp createdAt
    timestamp updatedAt
  }

  EDGE {
    string edgeId PK
    string parentId FK
    string childId FK
    number orderKey
    timestamp createdAt
    timestamp updatedAt
  }

  EVIDENCE {
    string evidenceId PK
    string type
    string title
    string url
    string ref
    string excerpt
    timestamp createdAt
  }
```

---

## Mermaid：Firestoreのパス構造（uid配下も明示）

```mermaid
flowchart TB
  U[users/{uid}] --> T[trees/{treeId}]
  T --> N[nodes/{nodeId}]
  T --> E[edges/{edgeId}]
  N --> EV[evidence/{evidenceId}]
```

---

## Mermaid：layout “触ったノードだけ保存” の流れ（更新版）

```mermaid
sequenceDiagram
  autonumber
  participant F as Frontend (ReactFlow)
  participant O as OrganizeService (Connect RPC)
  participant FS as Firestore

  F->>O: GetTree(treeId) + ListNodes/ListEdges (or GetSnapshot)
  O->>FS: read trees/{treeId}, nodes/*, edges/*
  FS-->>O: docs
  O-->>F: nodes + edges

  F-->>F: ELKレイアウト計算\n(※layoutLocked=trueのnodeは固定アンカー)

  Note over F: ユーザーがノードAをドラッグして「確定（lock）」した場合のみ保存
  F->>O: UpdateNodeFields(nodeId=A, layoutLocked=true, layout={x,y})
  O->>FS: update nodes/{A} {layoutLocked:true, layout:{x,y}}
  FS-->>O: ok
  O-->>F: Node(updated)

  Note over F: layoutLocked=falseのノードは保存せず、次回もELKで自動配置
```

---

## 共有RPCスキーマ（Proto）

フロントとバックエンドの共有型・RPC定義は以下を参照。

* `rpc/specs/proto-contracts.md`
