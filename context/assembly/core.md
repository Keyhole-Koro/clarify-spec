# Context Assembly Core 仕様

## 目的

`Context Assembly Layer` を read-only の文脈生成器として定義し、Actの毎ターン文脈生成を安定化する。

## スコープ / 非スコープ

* スコープ: 6段階パイプライン、責務境界、degrade
* 非スコープ: Organize書き込み、永続化、UI状態管理

## 前提・依存

* `context/assembly/bundle-schema.md`
* `context/model/topic-model.md`
* `organize/specs/pipeline/agents.md`

## 契約（I/O）

入力:

* `AssembleContextInput`
* Firestore/GCS read-only access

出力:

* `AssembleContextOutput`

## 正常フロー

1. Intent Resolver
2. Candidate Retrieval
3. Context Expansion
4. Ranking
5. Compression / Budgeting
6. Prompt Bundle Builder

### 段ごとの MUST

* Intent Resolver: `intent`, `focusNodeIds`, `retrievalPolicy` を決定
* Retrieval: focus/neighbor/evidence/recent_delta を収集
* Expansion: intent-dependent policy を適用
* Ranking: relation type を重みに含める
* Budgeting: token予算超過時に段階圧縮（L3->L2->L1）
* Builder: `PromptBundle + diagnostics` を返す

## 異常フロー（error/retryable/stage）

* `ASSEMBLY_VALIDATE_INPUT`: 入力不正
* `ASSEMBLY_RETRIEVE`: Firestore/GCS参照失敗（`retryable=true`）
* `ASSEMBLY_RANK`: 特徴量欠損（degrade継続）
* `ASSEMBLY_BUDGET`: 予算不足（degrade継続、`truncationReason=token_budget`）

## 数値パラメータ

* focus nodes: soft max 10, hard max 30
* neighbors (1-hop): max 8
* unresolved: max 3
* evidence: max 2 per focus (hard upper 5 total)

## 受け入れ条件（DoD）

* Assembly は Firestore/GCS へ書き込みしない
* intentごとに展開ポリシーが分岐している
* 予算超過時に deterministic に圧縮される
* diagnostics が必ず返る

## 実装メモ（最小）

* Retriever / Assembler / Renderer は分離する
* 「何を落としたか」を diagnostics に残す
* 取得失敗時は全体失敗より最小bundle degradeを優先する
