# Organize Guide（Human View）

## Organize は何をするか

`Organize` は、外部入力や Act で得た知見をそのまま捨てず、あとで再利用できる知識正本へ変換する write path である。

役割を一言で言うと、

* バラバラの入力を topic 単位の知識へ整理する
* 確定済みのノード、エッジ、要約、索引を更新する
* Act が読むための安定した材料を Firestore/GCS に供給する

`Act` が「今この場の探索」を担当するのに対して、`Organize` は「後で何度も使える形への定着」を担当する。

## Organize がしないこと

* ユーザーへの stream 応答を返さない
* Firestore/GCS を read するだけの query layer にはならない
* Act の `PromptBundle` を生成しない
* 実行中の ephemeral memory を保持しない

## どこに何を書くか

* Firestore:
  * topic, node, edge, version, index, evidence ref のような確定済み軽量メタ
* GCS:
  * 本文、draft、outline、map、rollup、bundle description のような重い実体
* Pub/Sub:
  * 各 Agent をつなぐ非同期イベント

```mermaid
flowchart LR
  IN[外部入力 / raw observation] --> A0[A0 MediaInterpreter]
  A0 --> A1[A1 Atomizer]
  A1 --> A2[A2 Router]
  A2 --> A3B[A3b Bundler]
  A3B --> A3[A3 Cleaner]
  A3B --> A6[A6 BundleDescription]
  A3 --> A4[A4 Indexer]
  A3 --> A7[A7 NodeRollup]
  A5[A5 Balancer] --> A7

  A0 --> GCS[(GCS)]
  A1 --> GCS
  A2 --> GCS
  A3 --> GCS
  A4 --> GCS
  A6 --> GCS
  A7 --> GCS

  A0 --> FS[(Firestore)]
  A1 --> FS
  A2 --> FS
  A3 --> FS
  A4 --> FS
  A5 --> FS
  A6 --> FS
  A7 --> FS
```

## 全体像

`Organize` は 1 本のイベントバス `mind-events` を中心に動く。  
ただし `mind-events` はインフラ名であり、知識モデル名ではない。知識モデルの正本用語は `topic`, `node`, `edge` である。

```mermaid
flowchart TD
  E[Event Bus<br>mind-events] --> A0[A0]
  E --> A1[A1]
  E --> A2[A2]
  E --> A3B[A3b]
  E --> A3[A3]
  E --> A4[A4]
  E --> A5[A5]
  E --> A6[A6]
  E --> A7[A7]

  A3 --> T[Topic Graph<br>topics/{topicId}/nodes, edges]
  A4 --> I[Index Items]
  A7 --> R[Node Rollup]
```

## 処理の順番

### 1. A0 MediaInterpreter

入力を受け取って、raw/extracted な実体を保存する入口。

* 受けるもの:
  * URL, HTML, GCS object など
* やること:
  * 入力を正規化する
  * 必要なら heavy extract に回す
  * Firestore `inputs/{inputId}` と GCS `mind/inputs/{inputId}.md` を作る
* 次へ渡すもの:
  * `input.received`

### 2. A1 Atomizer

入力本文を、あとで整理しやすい小さな単位に分解する。

* やること:
  * 抽出済み本文を atom に分ける
  * Firestore `atoms/{atomId}` と GCS `mind/atoms/{atomId}.md` を作る
* 次へ渡すもの:
  * `atom.created`

### 3. A2 Router

atom を今の topic draft に反映する。

* やること:
  * `latestDraftVersion` を CAS で進める
  * GCS に draft 本文を versioned で保存する
* 次へ渡すもの:
  * `draft.updated`

### 4. A3b Bundler

draft の増分を、そのまま Cleaner に投げる前に中間成果物へ束ねる。

* やること:
  * 今回増えた atom 群を `PipelineBundle` にまとめる
  * Firestore `pipelineBundles/{bundleId}` を作る
* 次へ渡すもの:
  * `bundle.created`

### 5. A6 BundleDescription

bundle を人間が読みやすい説明にする補助 Agent。

* やること:
  * bundle の説明 HTML を GCS に出す
  * Firestore `bundles/{bundleId}.descRef` を更新する
* 主目的:
  * デバッグしやすくする
  * bundle の中身を後から見返しやすくする

### 6. A3 Cleaner

`Organize` の中心。bundle を確定済み知識へ反映する。

* やること:
  * outline を更新する
  * `topics/{topicId}/nodes/*` を upsert する
  * `topics/{topicId}/edges/*` を upsert する
  * `latestOutlineVersion` を進める
* 次へ渡すもの:
  * `outline.updated`
  * `topic.node_changed`
  * `atom.reissued`

Cleaner は「未整理の断片」を「topic graph の正本」へ変える役割を持つ。

```mermaid
flowchart LR
  B[PipelineBundle] --> C[A3 Cleaner]
  C --> O[GCS Outline]
  C --> N[Firestore Nodes]
  C --> EG[Firestore Edges]
  C --> EV1[outline.updated]
  C --> EV2[topic.node_changed]
  C --> EV3[atom.reissued]
```

### 7. A4 Indexer

outline と graph から検索・ランキング用の索引を作る。

* やること:
  * `index_items/*` を更新する
  * map を GCS に出す
  * `latestMapVersion` を進める
* 使い道:
  * Act の ranking
  * topic 内探索
  * 重要ノード発見

### 8. A7 NodeRollup

変更された node ごとに、詳細本文とは別の注入用短要約を整える。

* やること:
  * node 単位の rollup HTML を GCS に出す
  * `topics/{topicId}/nodes/{nodeId}.rollupRef` を更新する
* 使い道:
  * Act の context assembly で短い要約を注入する
  * UI でノード詳細を見やすくする

### 9. A5 Balancer

graph 全体の偏りや未解決部分を見て、再整理を促す補助 Agent。

* やること:
  * metrics を見て是正 operation を作る
  * 必要なら `topic.node_changed` や `topic.metrics.updated` を再発行する
* 使い道:
  * 冗長ノードの是正
  * 偏りの大きい topic の調整
  * 未解決ノードの掘り起こし

## Organize と Act の関係

`Act` と `Organize` はつながっているが、役割は逆である。

```mermaid
flowchart LR
  U[User] --> ACT[Act]
  ACT -->|read-only| FS[(Firestore)]
  ACT -->|read-only| GCS[(GCS)]
  ACT -. proposal .-> ORG[Organize]
  ORG -->|write path| FS
  ORG -->|write path| GCS
```

* Act:
  * 今の質問に答える
  * read-only で context を集める
  * ephemeral memory を持つ
* Organize:
  * その場の候補を確定済み知識へ変える
  * write path を担当する
  * at-least-once でも壊れないよう冪等に動く

## Organize が Act に供給するもの

Act が読みたいのは生の入力ではなく、整理済みの知識である。

| 供給物 | 主担当 | 何に使うか |
| --- | --- | --- |
| `context_summary` | A3 / A7 | node の短い文脈注入 |
| `evidence_refs` | A0 / A1 / A3 | grounding |
| `relation_importance` | A3 / A4 | ranking |
| `recent_delta` | A2 / A3 | 最近の変化 |
| `confidence/quality` | A4 / A5 | 優先順位づけ |

## なぜ Agent を分けるのか

理由は 3 つある。

* 書き込み責務を段階ごとに切り分けたい
* 重い処理と軽い処理を分離したい
* 失敗しても topic 全体を壊さず再試行したい

1 つの巨大なジョブにすると、draft 更新、graph 更新、索引更新、rollup 更新のどこで壊れたか追いにくい。  
Agent 分割にすると、イベントごとに retry, DLQ, ledger, lease を適用しやすい。

## どうやって壊れにくくするか

```mermaid
sequenceDiagram
  autonumber
  participant PS as Pub/Sub
  participant AG as Agent
  participant EL as event_ledger
  participant LS as lease
  participant DB as Firestore/GCS

  PS->>AG: message
  AG->>EL: reserve(idempotencyKey)
  EL-->>AG: started or duplicate
  AG->>LS: acquire(topic or node)
  LS-->>AG: ok or retry
  AG->>DB: compare-and-set write
  DB-->>AG: success or stale
  AG->>PS: ack / retry / DLQ
```

### 1. Idempotency

同じイベントが重複して届いても、`event_ledger` で二重処理を避ける。

### 2. Lease

同じ topic や node を複数 worker が同時に更新しないようにする。

### 3. CAS

`latestDraftVersion` や `latestOutlineVersion` を更新するときは、期待していた版と一致する場合だけ書く。

### 4. DLQ

永続的に失敗するイベントは Dead Letter Queue に流し、後で調べられるようにする。

## Organize の成果物

最終的に `Organize` が残すのは次のようなもの。

* Firestore:
  * `topics/{topicId}`
  * `topics/{topicId}/nodes/{nodeId}`
  * `topics/{topicId}/edges/{edgeId}`
  * `index_items/*`
  * `pipelineBundles/{bundleId}`
  * `organizeOps/{opId}`
* GCS:
  * `mind/inputs/...`
  * `mind/atoms/...`
  * `mind/drafts/...`
  * `mind/outlines/...`
  * `mind/maps/...`
  * `mind/node_rollup/...`
  * `mind/bundle_desc/...`

## どこから読むべきか

### 最短で把握

1. このファイル
2. `organize/specs/pipeline/summary.md`
3. `organize/specs/pipeline/agents.md`

### 厳密に詰める

* 中核規約:
  * `organize/specs/pipeline/core.md`
* Agent 契約:
  * `organize/specs/pipeline/agents.md`
* 個別 Agent:
  * `organize/agents/a3-cleaner.md`
  * `organize/agents/a7-node-rollup.md`
  * `organize/agents/a5-balancer.md`

## ひとことでまとめる

`Organize` は、断片的な入力を topic graph と versioned 文書へ整理し、Act が何度でも読める知識正本へ育てる非同期 write pipeline である。
