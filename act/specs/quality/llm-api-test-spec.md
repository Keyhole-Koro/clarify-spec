# LLM API 呼び出しテスト仕様（Vertex AI / Gemini）

## 目的

Act backend（Go）の外部LLM呼び出しが、モデル切替・grounding・thought stream・deep researchを正しく扱えることを検証する。

## 対象

* Gemini 3 Flash 呼び出し
* Web Grounding有効呼び出し
* includeThoughts有効呼び出し
* Deep Research（Interactions API）

## テストレイヤ

1. Contract test（スタブ）
2. Integration test（実API / 低頻度）

## 必須ケース（MUST）

1. Flash基本応答
* `LLM_PROFILE_GEMINI_3_FLASH` で正常応答
* textが `stream_parts[].thought=false` に流れる

2. Grounding有効
* `grounding_config.use_web_grounding=true`
* groundingメタデータが取得できる

3. Thought stream有効
* `thinking_config.include_thoughts=true`
* `stream_parts[].thought=true` が1件以上返る

4. Deep Research背景実行
* `research_config.use_deep_research=true`
* `in_progress -> completed` の遷移を追える

5. エラーマッピング
* 429/503/timeout を `UNAVAILABLE` / `DEADLINE_EXCEEDED` へ変換

## 実行方針

* スタブテストは毎PRで実行
* 実APIテストは日次または手動（コスト制御）
* 失敗時は `traceId` / `profile` / `grounding` / `thinking` をログ出力

## 完了条件（DoD）

* 5ケースが定義済みで、実装時に自動化できる
* 本番切替前に実APIの最低3ケースが通る
