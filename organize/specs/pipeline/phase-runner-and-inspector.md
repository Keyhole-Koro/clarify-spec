# Organize Phase Runner / Inspector 仕様

Version: 1.0

目的: Organize の各 phase を Pub/Sub から切り離して単体実行し、入出力と副作用 preview を確認できるようにする。

## 1. 目的

この仕組みで実現したいこと:

* Agent を phase ごとに単体実行する
* fixture を差し替えて結果を比較する
* Firestore / GCS write を preview する
* emit event を確認する
* golden との差分を見る

## 2. 対象 phase

少なくとも次を対象とする。

* `A0 MediaInterpreter`
* `A1 Atomizer`
* `TopicResolver`
* `A2 DraftAppender`
* `A3b Bundler`
* `A6 BundleDescription`
* `A3 Cleaner`
* `A4 Indexer`
* `A7 NodeRollup`
* `A5 Balancer`

## 3. 基本原則

* Agent の core logic は transport 層から分離する
* Pub/Sub push handler は薄い adapter にとどめる
* phase runner は core logic を直接呼ぶ
* inspector は write preview と emit preview を可視化する

## 4. Phase Runner

### 4.1 役割

phase runner は 1 Agent を 1 回だけ直接実行する CLI である。

共通 I/O は `organize/specs/pipeline/run-phase-contract.md` を正本とする。

入力:

* fixture file
* 実行対象 phase
* 実行モード

出力:

* normalized output
* emitted events
* Firestore write preview
* GCS write preview
* diagnostics

### 4.2 例

```bash
run-phase a1 --input fixtures/a1/multi-claim.md
run-phase topic-resolver --input fixtures/topic-resolution/similar-topics.json
run-phase a3 --input fixtures/a3/bundle-created.json
```

### 4.3 実行モード

* `preview`
  * write を実際には行わない
* `apply-local`
  * emulator に対して write する
* `golden-check`
  * expected と比較する
* `update-golden`
  * expected fixture を更新する

## 5. Inspector

### 5.1 役割

inspector は phase 実行結果を人間が見やすい形で表示する。

最低限表示するもの:

* input
* normalized result
* emitted events
* Firestore write preview
* GCS write preview
* expected との差分

### 5.2 形

first pass は CLI でよい。

将来的には HTML または小さな local web UI を持ってよい。

## 6. Fixture 構成

最小セットは `organize/specs/pipeline/minimal-fixture-set.md` を正本とする。

```text
fixtures/
  a0/
  a1/
  topic-resolution/
  a2/
  a3b/
  a6/
  a3/
  a4/
  a7/
  a5/
```

各 fixture は最低限次を持つ。

* `input.*`
* `expected.json`
* `notes.md`（任意）

## 7. Expected Output の形

`expected.json` には少なくとも次を持つ。

* `result`
* `events`
* `firestoreWrites`
* `gcsWrites`

`preview` では path と payload shape が分かればよい。

## 8. Firestore / GCS Preview

runner は副作用を直接実行する前に preview object を作る。

Firestore preview:

* `path`
* `operation`: `create | update | upsert`
* `data`

GCS preview:

* `path`
* `contentType`
* `bodyPreview`
* `sha256`

## 9. A3 Cleaner の特別扱い

A3 は複雑なので 3 段に分けて preview 可能にする。

* `candidate-generation`
* `entity-resolution`
* `graph-commit-preview`

MUST:

* commit 前に node/edge diff を見られるようにする
* `atom.reissued` 候補を明示する

## 10. Golden Test 方針

最低限の golden を持つ。

* A1: claim 数、reject 数、kind
* TopicResolver: attach / create_new
* A3b: bundle shape
* A3: node / edge / outline diff
* A7: contextSummary / detailHtml

MUST:

* golden は deterministic input で比較可能
* LLM 実呼び出しを使う場合は stub mode も用意する

## 11. LLM Stub Mode

runner 用に LLM stub mode を用意する。

目的:

* local test の再現性を上げる
* cost を抑える
* golden diff を安定させる

モード:

* `real-llm`
* `stub-llm`

## 12. 実装境界

推奨構成:

* `src/core/`
  * pure or near-pure logic
* `src/adapters/pubsub/`
  * envelope / push handler
* `src/adapters/firestore/`
* `src/adapters/gcs/`
* `src/devtools/phase-runner/`
* `src/devtools/inspector/`

## 13. 受け入れ条件

* 各 phase を Pub/Sub なしで 1 回実行できる
* preview mode で副作用なしに出力確認できる
* golden-check で差分が見える
* A3 の commit 前 preview が見える
* fixture の更新手順が明確である
