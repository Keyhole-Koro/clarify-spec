# AI実装指示テンプレート（Phase 1〜3 / 自然言語版）

このファイルは、実装時にLLMへ渡す指示テンプレート集である。
コードフェンスは使わず、要件を箇条書きで固定する。

共通で添える参照:

* `frontend/frontend-spec.md`
* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/act-flow.md`

---

## Prompt: Phase 1（キャンバス土台 + ストリーミング）

### 依頼文（そのまま渡す）

* あなたは Next.js / ReactFlow / Connect RPC の実装担当。
* 参照仕様を厳守して、Phase 1 を実装すること。

### 参照仕様

* `frontend/frontend-spec.md`
* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/act-flow.md`

### 不変ルール

1. SPA（App Router）として動作
2. ReactFlowはClient Component、必要ならSSR回避を行う
3. ZustandでBlock状態を管理し、PatchOpを逐次適用
4. PatchOpは `upsert` と `append_md` のみ
5. `append_md` は `contentMd` 追記として処理

### 作業対象

1. `src/features/knowledgeTree/store.ts`
2. `src/features/actExplore/hooks/useActStream.ts`
3. `src/features/graph/components/CustomNode.tsx`
4. `src/features/graph/components/GraphCanvas.tsx`
5. `src/components/ui/AskForm.tsx`

### 出力要件

* 上記5ファイルの実装コードまたは差分
* 最小限の型定義（Block/PatchOp）
* ReactFlowのnodes/edges変換ロジック
* 送信 -> ストリーム受信 -> applyPatch の一連処理

---

## Prompt: Phase 2（複数選択 + コンテキスト送信 + 依存エッジ）

### 依頼文（そのまま渡す）

* Phase 1 実装済みコードを前提に、Phase 2 を追加すること。

### 参照仕様

* `frontend/frontend-spec.md`
* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/act-flow.md`

### 不変ルール

1. `parent_id` 構造エッジと `contextEdges` は分離
2. Ask送信時は選択ノード本文を同送
3. 新規ACT_ROOT受信時に依存エッジを追加

### 作業対象

1. `src/features/knowledgeTree/store.ts`
2. `src/features/graph/components/GraphCanvas.tsx`
3. `src/features/graph/components/CustomNode.tsx`
4. `src/components/ui/AskForm.tsx`
5. `src/features/actExplore/hooks/useActStream.ts`

### 出力要件

* `selectedNodeIds` / `contextEdges` 追加
* `onSelectionChange` によるストア同期
* 送信後に `selectedNodeIds` クリア
* 選択中ノードの視覚ハイライト

---

## Prompt: Phase 3（右ペイン詳細 + run_act分岐）

### 依頼文（そのまま渡す）

* Phase 1/2 実装済みコードを前提に、Phase 3 を追加すること。

### 参照仕様

* `frontend/frontend-spec.md`
* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/act-flow.md`

### 不変ルール

1. 右ペインは `MarkdownPane` 中心
2. `actions[].on_click.type === run_act` を実行可能にする
3. run_act時の `anchor` に起点ノードIDを設定

### 作業対象

1. `src/features/knowledgeTree/store.ts`
2. `src/features/nodeMarkdown/components/MarkdownPane.tsx`
3. `src/features/graph/components/GraphCanvas.tsx`
4. `src/features/graph/components/CustomNode.tsx`
5. `src/features/actExplore/hooks/useActStream.ts`

### 出力要件

* `activeNodeId` 追加
* ノードクリック/ダブルクリックで `activeNodeId` 更新
* MarkdownPaneで詳細表示（sanitize必須）
* run_actから新規ストリーム開始と依存エッジ接続

---

## レビュー観点

* `append_md` が追記になっているか
* ReactFlow変換で親子構造と依存関係が崩れていないか
* run_act時のanchor設定が漏れていないか
* Storeの責務がUI状態に限定されているか
