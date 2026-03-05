# RunAct 実装仕様（LangGraph）

## 目的

`ActService.RunAct` を仕様どおりの server-streaming として実装し、フロントへ安定した Draft Patch を供給する。

## 境界

* 入力: `RunActRequest`
* 出力: `RunActEvent` stream
* 非スコープ: Firestore永続化、layout固定、Organize更新
* 実装言語: Go（バックエンド）

## 前提仕様

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/contracts/gemini-vertex-response-schemas.md`
* `act/specs/behavior/act-langgraph-runtime.md`
* `act/specs/behavior/act-flow.md`
* `act/specs/quality/backend-parameter-index.md`

## モデル方針（Vertex AI）

* 既定プロファイル: `GEMINI_3_FLASH`
* 深掘りプロファイル: `GEMINI_DEEP_RESEARCH`
* Web検索根拠が必要な場合は `use_web_grounding=true` を有効化
* `llm_config.profile` 未指定時はサーバ側で `GEMINI_3_FLASH` を採用
* thought配信が必要な場合は `thinking_config.include_thoughts=true`
* Deep Researchは Interactions API 前提で `research_config.use_deep_research=true`

## 状態遷移（MUST）

1. `ValidateRequest`
2. `LoadContext`
3. `BuildPrompt`
4. `GenerateWithModel`
5. `NormalizePatchOps`
6. `EmitStream`
7. `Finalize`
8. `HandleError`

## 実装ルール（MUST）

* `PatchOp` は `upsert` / `append_md` のみ許可
* `done` と `error` は同一Runで排他
* stream終端は最後に1回だけ
* `ACT_TYPE_UNSPECIFIED` は `INVALID_ARGUMENT`
* 親 -> 子の順序で patch を送る
* `append_md` の対象未存在時は `upsert` 補完を先行
* Vertex AI 呼び出しは `llm_config` と `grounding_config` を反映
* thoughtトークンは `stream_parts[].thought=true` で送出
* Go実装は goroutine/chan で stream backpressure を制御

## 制限値（MUST）

* request timeout: 90s
* model retry: 2回
* 推論ループ上限: 5
* patch_ops上限: 400/run
* anchor_node_ids: <= 40
* context_node_ids: <= 120
* thought stream flush interval: <= 500ms

## エラーハンドリング

* 入力不正: 即 `error` 終端
* 外部依存障害: 再試行後 `error` 終端
* 部分送信後の破綻: `error` で終端（`done` は返さない）
* Deep Research timeout時は通常Flashへフォールバック可能（設定で制御）

## 完了条件（DoD）

* 正常入力で `done=true` まで返る
* 異常入力で `INVALID_ARGUMENT` を返す
* 禁止op混入が発生しない
* thoughtが有効時、`stream_parts.thought` が1回以上返る
