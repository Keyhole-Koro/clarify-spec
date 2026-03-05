# Organize Agent 仕様（I/O + 競合対策）

以下、既存フローを保ちつつ「MUST」を追加した確定版。

詳細仕様（Agent別）:

* `organize/agents/a0-media-interpreter.md`
* `organize/agents/a1-atomizer.md`
* `organize/agents/a2-router.md`
* `organize/agents/a3b-bundler.md`
* `organize/agents/a3-cleaner.md`
* `organize/agents/a4-indexer.md`
* `organize/agents/a5-balancer.md`
* `organize/agents/a6-bundle-description.md`
* `organize/agents/a7-node-rollup.md`

---

## A0 MediaInterpreterAgent（メディア→テキスト化）

### Input event

* `type = media.received`
* payload（既存どおり）

### Output

* GCS：`mind/inputs/{inputId}.md`
* Firestore：`inputs/{inputId}`（status/refs）

### Emits

* `type = input.received`（A1へ）payload: `{ inputId }`

### 競合対策（MUST）

* ledger：`type:media.received/inputId:{inputId}` を確保
* inputs/{inputId}.textRef.sha256 が同一で status=processed ならスキップ可能（冪等）

---

## A1 AtomizerAgent（Input→Atom抽出）

### Input

* `type = input.received` payload: `{ inputId }`

### Output

* GCS：`mind/atoms/{atomId}.md`
* Firestore：`atoms/{atomId}`（複数）

### Emits

* `type = atom.created` payload: `{ inputId, atomIds }`

### 競合対策（MUST）

* ledger：`type:input.received/inputId:{inputId}`
* atoms生成は「同一inputIdの再処理」で二重生成しないようにする：

  * 推奨：`atoms` に `sourceInputId` を持たせ、同inputIdで既に atoms が存在するなら再生成しない
  * もしくは atomId を deterministic（inputId+span+typeのhash）にする

---

## A2 RouterAgent（Atom→Draft追記）

### Input

* `type = atom.created` payload: `{ inputId, atomIds }`

### Output

* Firestore：`drafts/{outlineId}` 更新（latestDraft, lastAppendedAtomIds）
* GCS：`mind/drafts/{outlineId}/v{n}.md`（追記後）

### Emits

* `type = draft.updated` payload: `{ outlineId, draftVersion, appendedAtomIds }`

### 競合対策（MUST）

* ledger：`type:atom.created/inputId:{inputId}`（または atomIdsハッシュ込み）
* lease：`outline:{outlineId}`（必須）
* CAS：`drafts/{outlineId}.latestDraft.version` を expectedPrev として transaction 更新

  * 成功したら `draftVersion = prev+1` を emit
  * CAS失敗 → ACKして終了（別ワーカーが先に更新済み）

---

## A3b BundlerAgent（Draft差分→Bundle生成）

### Input

* `type = draft.updated` payload: `{ outlineId, draftVersion, appendedAtomIds }`

### Output

* Firestore：`bundles/{bundleId}` create

### Emits

* `type = bundle.created` payload: `{ outlineId, bundleId }`

### 競合対策（MUST）

* ledger：`type:draft.updated/outlineId:{outlineId}/draftVersion:{draftVersion}`
* （推奨）lease：`outline:{outlineId}`（Bundlerは順序/差分依存が強いので）
* 冪等：同じ (outlineId, draftVersion) から bundle を二重作成しない

  * 推奨：`bundles` に `sourceDraftVersion` を持たせ、同版が既にあればスキップ

---

## A6 BundleDescriptionAgent（Bundle→HTML説明）

### Input

* `type = bundle.created` payload: `{ outlineId, bundleId }`

### Output

* GCS：`mind/bundle_desc/{bundleId}/v{n}.html`（+ css）
* Firestore：`bundles/{bundleId}.descRef` 更新

### Emits（任意）

* `type = bundle.described`

### 競合対策（MUST）

* ledger：`type:bundle.created/bundleId:{bundleId}/purpose:desc`（purposeを分ける）
* version(CAS)：`descRef.version` を prev+1 更新（既に同versionならスキップ）
* **statusは分離**（後述）

---

## A3 CleanerAgent（Bundle→Outline/MindTree反映）

### Input

* `type = bundle.created` payload: `{ outlineId, bundleId }`

### Output

* GCS：`mind/outlines/{outlineId}/v{n}.md`
* Firestore：

  * `outlines/{outlineId}.latestOutline` 更新
  * `mindtree_nodes/*`, `mindtree_edges/*` upsert

### Emits

* `type = outline.updated` payload: `{ outlineId, outlineVersion }`
* `type = mindtree.node_changed` payload: `{ outlineId, nodeId, reason }`（※後述：1イベント=1node推奨）
* `type = atom.reissued`（話題違いの再投入）

### 競合対策（MUST）

* ledger：`type:bundle.created/bundleId:{bundleId}/purpose:apply`
* lease：`outline:{outlineId}`（必須）
* 冪等（必須）：bundleの二重適用を必ず防ぐ

  * 推奨：`bundles/{bundleId}.appliedAt` を transaction で “未適用なら適用済みにする” をCASし、適用はその後
  * または `mindtree_edges/{bundleId}_{edgeId}` のように bundleId をキーにして upsert（増殖しない）
* outlineVersion(CAS)：`outlines.latestOutline.version` を prev+1 更新。CAS失敗はACKでスキップ

---

## A4 IndexerAgent（Outline→Index/Map）

### Input

* `type = outline.updated` payload: `{ outlineId, outlineVersion }`

### Output

* Firestore：`index_items/*` upsert
* GCS：`mind/maps/{outlineId}/v{n}.md`
* Firestore：`outlines/{outlineId}.latestMap` 更新

### 競合対策（MUST）

* ledger：`type:outline.updated/outlineId:{outlineId}/outlineVersion:{outlineVersion}`
* lease：`outline:{outlineId}`
* 古い版スキップ：`outlineVersion < latestOutline.version` はACKして終了

---

## A7 NodeRollupAgent（Node rollup）

### Input

* `type = mindtree.node_changed` または `node.rollup_requested`

### Output

* GCS：`mind/node_rollup/{nodeId}/v{n}.html`
* Firestore：`mindtree_nodes/{nodeId}.rollupRef` + `rollupWatermark`

### Emits（任意）

* `type = node.rollup_updated`

### 競合対策（MUST）

* **イベント形の推奨変更（重要）**：`mindtree.node_changed` は `nodeId` を単数にする

  * payload推奨：`{ outlineId, nodeId, reason }`
* ledger：`type:mindtree.node_changed/nodeId:{nodeId}/generation:{gen}`（generationを含めると強い）
* lease：`node:{nodeId}`
* watermarkスキップ：要求が `rollupWatermark` 以下ならACKして終了（古い要求の無害化）

---

## A5 BalancerAgent（偏り是正）

### Input

* `type = mindtree.metrics.updated`（または定期実行）

### Output

* Firestore：OperationPatchログ（推奨）

  * `organizeOps/{opId}`

### Emits

* `mindtree.node_changed` / `mindtree.metrics.updated`

### 競合対策（MUST/SHOULD）

* 変更適用は lease（outlineまたはscopeNodeId）推奨
* patch適用は generation(CAS) 推奨

---
