# Act フロー仕様（Draft -> Review -> Commit）

## 1. Act UI 状態遷移

```mermaid id="act_state_01"
stateDiagram-v2
  [*] --> Idle

  Idle --> Configure: Act起動\n種類とアンカー選択
  Configure --> Running: RunAct開始\nstream
  Running --> DraftReady: patch適用\nドラフト生成
  DraftReady --> Running: 追加の指示\n続けて質問
  DraftReady --> Review: 提案の比較と選別
  Review --> Commit: ApplyPatch\n選択分だけ永続化
  Commit --> Done: Firestore反映\nsnapshot更新
  Done --> Idle: 閉じる 次のAct

  Running --> Error: 失敗 キャンセル
  Error --> Configure: 再試行
```

決めポイント

* Actの結果は Draft インメモリ にだけ反映
* ユーザーが Commit したときだけ organize に書き込む

---

## 2. シーケンス（Frontend -> ActService -> Firestore -> LLM）

```mermaid id="act_sequence_01"
sequenceDiagram
  autonumber
  participant U as User
  participant F as Frontend Act UI plus ReactFlow
  participant A as ActService Connect RPC
  participant O as OrganizeService Connect RPC
  participant FS as Firestore organize
  participant L as LLM Geminiなど

  %% prepare
  U->>F: Act起動\nactType anchors message
  F->>A: RunAct treeId actType anchorNodeIds userMessage\nplus auth

  %% context build
  A->>FS: read nodes edges around anchors\nk hop evidence optional
  FS-->>A: context snapshot

  %% generation
  A->>L: prompt context userMessage constraints\nedge schema layout policy etc
  L-->>A: streaming output

  %% streaming back
  loop stream
    A-->>F: RunActEvent.text_delta 任意
    A-->>F: RunActEvent.patch_ops\nUpsertNode UpsertEdge UpdateNodeFields AddEvidence
    F-->>F: apply patch to DraftGraph in-memory
  end

  %% review
  U->>F: 提案を選ぶ\n全部 一部
  F->>O: ApplyPatch treeId selected_ops\nplus mutation_id

  %% persist
  O->>FS: batch write\nnodes edges evidence\norderKey採番
  FS-->>O: ok
  O-->>F: ApplyPatchResponse changed ids

  %% layout
  F-->>F: ELKレイアウト\nlayoutLocked trueは固定\n新規ノードは未lockedで自動配置
  F-->>U: organize反映完了
```

決めポイント

* RunAct は 読むだけ Firestore書き込みなし
* 永続化は ApplyPatch のみ OrganizeService
* layoutは lockedノードだけ保存 それ以外はELKで馴染ませる

---

## 3. Draft Patch の種類（edge方式）

```mermaid id="act_patch_ops_01"
flowchart TB
  subgraph Draft[Draft in-memory]
    N1[UpsertNode\ntitle contentMd]
    E1[UpsertEdge\nparentId childId orderKey optional]
    U1[UpdateNodeFields\nlayoutLocked layout\n基本Actは触らない]
    V1[AddEvidence\nnodeId type url or ref excerpt]
  end

  subgraph Commit[Commit ApplyPatch to Firestore]
    CN[write nodes nodeId]
    CE[write edges edgeId]
    CV[write nodes nodeId evidence evidenceId]
  end

  N1 --> CN
  E1 --> CE
  U1 --> CN
  V1 --> CV

  note1[orderKeyはCommit時に決める\n親配下max plus 1024 など]
  E1 -.-> note1
```

おすすめ運用 ここで固定

* Actが orderKey を直接決めない Draftでは空でもOK
  -> Commit側で 親配下の末尾に追加 指定位置 を見て採番
* Actが layout を勝手に触らない
  -> layoutLocked layout は ユーザー操作のみ

---

## 4. フォローアップ時の扱い（Draftを文脈に含める）

```mermaid id="act_followup_01"
flowchart LR
  A[Firestore Snapshot\n現在のorganize] --> P[Act Prompt Context]
  D[DraftGraph\n未commitの提案] --> P
  U[User message] --> P
  P --> L[LLM]
  L --> R[patch_ops stream]
  R --> D
```

---

## 5. PatchOp表現の対応（UI Composerとの整合）

フロント側で扱う `PatchOp` は、UI Composer表現と内部表現のどちらでもよい。
ただし、適用時の意味は以下で固定する。

* UI Composer `upsert`（`blocks[]`）= 内部 `UpsertNode` / `UpsertEdge` 群
* UI Composer `append_md`（`block_id`, `delta`）= 対象ノード `contentMd` への追記
* `append_md` はストリーミング描画専用で、順次適用を前提とする
* Commit時の永続化は従来どおり `ApplyPatch` のみ

---

## 6. Backend実行仕様（LangGraph）

`RunAct` の内部状態遷移は `act/backend/act-langgraph-spec.md` を正本とする。
テスト観点は `act/backend/act-e2e-test-plan.md` を参照する。
