# ADK Worker Test Spec

## 目的

`act-adk-worker` の最小テスト仕様を固定し、stream 契約、error mapping、metadata 正規化、budgeting が実装差で崩れないようにする。

## スコープ / 非スコープ

* スコープ: worker 単体および worker entrypoint に対する最小ゴールデンケース
* 非スコープ: frontend E2E、Organize write path、長時間負荷試験

## 前提 / 参照

* `act/act-adk-worker/specs/act-adk-runtime.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/ai-streaming-cases.md`
* `act/act-api/specs/reliability/error-mapping-matrix.md`

## テストレイヤ

### 1. Unit

対象:

* intent 判定
* `RankAndBudget`
* error mapping
* metadata normalize

目的:

* pure function に近いロジックを早く壊れにくく検証する

### 2. Stream Golden

対象:

* `RunActEvent` 列
* terminal 種別
* patch op 順序
* metadata shape

目的:

* 出力順序と外部契約 shape を fixture 比較で固定する

### 3. Integration

対象:

* worker entrypoint
* request から stream 完了まで

目的:

* request -> assembly -> model stub -> normalize -> emit の一連経路を検証する

## 必須ゴールデンケース

### 1. intent

入力:

* 通常の探索入力
* intent 未判定入力

期待:

* 期待した intent / profile が選ばれる
* intent 未判定時は `explore` へフォールバックする

### 2. budget超過

入力:

* context 候補が token budget を超える request

期待:

* `RankAndBudget` が圧縮を行う
* drop 理由が diagnostics に残る
* stream は継続できる

### 3. error mapping

入力:

* validation error
* assembly retrieve failure
* model timeout / unavailable
* patch normalize failure

期待:

* `code/stage/retryable` が期待どおりにマッピングされる

### 4. stream順序

入力:

* thought, answer, patch, metadata を含む正常系 request

期待:

* `upsert` が `append_md` より先に出る
* 基本順序 `thought -> answer -> upsert -> append_md -> metadata -> terminal` を守る
* `done/error` は最後
* duplicate terminal が出ない

### 5. metadata正規化

入力:

* grounding metadata と tool metadata を含む request

期待:

* grounding metadata が `references[]` shape に正規化される
* tool metadata が `tool_name/status/summary` shape に正規化される
* raw SDK payload 全量が外部契約へ漏れない

## 最小 fixture セット

* 正常系: `flash_success`
* 正常系: `deep_research_fallback_success`
* 異常系: `validation_error`
* 異常系: `model_timeout`
* 異常系: `normalize_error`

## 検証観点（MUST）

* `PatchOp` が `upsert/append_md` のみ
* `done/error` 排他
* 終端後 event が来ない
* `append_md` が対応 block の `upsert` より先行しない
* metadata が最小 shape に正規化される
* `trace_id` が追跡可能

## 完了条件（DoD）

* `intent`, `budget超過`, `error mapping`, `stream順序`, `metadata正規化` の5ケースが自動検証できる
* unit / stream golden / integration の3層で最低1本以上が存在する
* 契約変更時に fixture 差分で破壊的変更を検知できる
