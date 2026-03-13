# E2E テスト戦略（最小）

## 目的

ローカルおよび本番前確認で最低限の品質を担保するため、壊れやすい経路を自動検証する。

## 対象

* `ActService.RunAct`
* フロントのstream購読ロジック（最小）
* Vertex AI API 呼び出し層（Gemini）

詳細なLLM API試験は `act/specs/quality/llm-api-test-spec.md` を参照。
ユースケース導線は `act/specs/usecases/README.md` を参照。
worker 単体の最小仕様は `act/act-adk-worker/specs/adk-worker-test-spec.md` を参照。

## 必須シナリオ

1. 正常系: patchが流れ `done` 終端
2. 途中失敗: patch途中で `error` 終端
3. 入力不正: 初手 `INVALID_ARGUMENT`
4. thought有効: `stream_parts[].thought=true` が返る
5. grounding有効: groundingメタデータを内部に保持し応答へ反映できる

## 検証観点（MUST）

* `PatchOp` が `upsert/append_md` のみ
* `done/error` 排他
* 終端後イベントが来ない
* traceIdが全ケースで取得できる
* Vertex API失敗時に期待コードへマッピングされる

## 実行方針

* CIで軽量3本を毎回実行
* 重い回帰（長文・高負荷）は手動回帰に分離
* 失敗時はシナリオ名とtraceIdを必ず出力
* LLM API呼び出しは本番/スタブの2レイヤでテストする

## Usecase対応

* `UC-ASK-EMPTY-01` -> `E2E-STREAM-SUCCESS`
* `UC-ASK-CONTEXT-01` -> `E2E-CONTEXT-EDGE-LINK`
* `UC-RUNACT-NODE-01` -> `E2E-RUNACT-ANCHOR-LINK`
* `UC-THINK-STREAM-01` -> `E2E-THOUGHT-STREAM`
* `UC-DEEP-FALLBACK-01` -> `E2E-DEEPRESEARCH-FALLBACK`

## 完了条件（DoD）

* 3シナリオが自動実行可能
* 回帰で失敗原因を再現できるログが残る
