# Context Assembly Execution Profile（Act ADK Worker）

## 目的

`ResolveIntent` / `RetrieveContext` / `RankAndBudget` を実装可能な固定値で定義し、RunAct 実行時の揺れをなくす。

## スコープ / 非スコープ

* スコープ: 判定優先順位、取得条件、重み、予算削減順、Firestore query/index
* 非スコープ: ADK worker内部のプロンプト文面

## 前提・依存

* `context/assembly/implementation.md`
* `context/assembly/core.md`
* `act/specs/quality/backend-parameter-index.md`
* `firestore/indexes.md`

## 契約（I/O）

入力:

* `RunActRequest`（`topic_id`, `user_message`, `context_node_ids`, `act_type`）
* frontend tool 由来の explicit UI context（任意）

出力:

* `AssembleContextOutput`（`bundle`, `diagnostics`）

## 正常フロー

1. `ResolveIntent`
2. `RetrieveContext`
3. `RankAndBudget`
4. `bundle + diagnostics` を返し、同一 worker 実行内で `GenerateWithModel` へ渡す

## Draft / Persisted 競合ルール

* frontend tool 由来の context は `explicit_ui_context` として persisted retrieval と別系統で扱う
* `explicit_ui_context` は `source`, `draft_revision`, `persisted_revision` を保持できる
* conflict 時は明示 UI 入力を優先する
* persisted retrieval は補助文脈として残し、bundle 内で `user_selected_draft_context` と `retrieved_persisted_context` を分離する
* worker は provenance を失ったまま draft と persisted をマージしてはならない

## ResolveIntent 固定ルール

### 優先順位（高 -> 低）

1. `run_act` action payload の明示指定
2. `act_type` (`explore|consult|investigate`)
3. キーワード分類（`比較|根拠|要約|整理`）
4. `context_node_ids` の有無
5. デフォルト `explore`

### 競合時ルール

* `ground` キーワードが含まれる場合は他候補より優先
* `compare` と `summarize` が競合する場合は `compare`
* 推定不能時は `explore`

## RetrieveContext 固定ルール

### 取得上限

* focus: soft最大 10、hard最大 30
* neighbor: 最大 8
* evidence: focus あたり最大 2、全体最大 5
* unresolved: 最大 3

### relation 優先

`contradicts > supports > depends_on > related_to`

### Firestore query（固定）

1. focus nodes
* collection: `workspaces/{workspaceId}/topics/{topicId}/nodes`
* where: `node_id in context_node_ids`（指定時）
* fallback: `orderBy(updated_at, desc).limit(2)`

2. neighbors
* collection: `.../edges`
* where: `from_node_id in focus_ids`
* orderBy: `weight desc, updated_at desc`
* limit: 30（後段で 8 へ圧縮）

3. evidence refs
* collection: `.../evidences`
* where: `node_id in focus_ids`
* orderBy: `confidence desc, updated_at desc`
* limit: 20（後段で 5 へ圧縮）

4. recent deltas
* collection: `.../actRuns`
* orderBy: `created_at desc`
* limit: 5

### 必要 index

* `nodes(topic_id, updated_at desc)`
* `edges(from_node_id, weight desc, updated_at desc)`
* `evidences(node_id, confidence desc, updated_at desc)`
* `actRuns(topic_id, created_at desc)`

## RankAndBudget 固定ルール

### Node score

`score = focus*0.30 + relation*0.20 + recency*0.15 + confidence*0.15 + selected_boost*0.10 + unresolved_boost*0.10`

### Evidence score

`score = linked_focus*0.35 + source_quality*0.20 + freshness*0.20 + contradiction*0.15 + citation_reuse*0.10`

### Token budget 削減順

1. related non-focus を削減
2. evidence 下位を削減
3. unresolved を削減
4. focus を L3->L2->L1 に圧縮（削除しない）

## 異常フロー（error/retryable/stage）

* 判定不能: `INTERNAL`, `retryable=true`, `stage=ASSEMBLY_RANK`（`explore` で継続）
* query失敗: `UNAVAILABLE`, `retryable=true`, `stage=ASSEMBLY_RETRIEVE`
* 予算圧縮失敗: `INTERNAL`, `retryable=true`, `stage=ASSEMBLY_BUDGET`（L1最小bundle）

## 数値パラメータ

* `context_node_ids_max=120`
* `load_hop=2`
* `load_nodes_max=500`
* `load_edges_max=1000`
* `load_evidence_max=1000`

## 受け入れ条件（DoD）

* 同一入力で意図判定が再現可能
* 同一入力で候補取得順序が再現可能
* budget超過時に `diagnostics.truncationReason` が残る
