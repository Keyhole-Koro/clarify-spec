# Frontend Stream 統合仕様

## 目的

`RunActEvent` をフロントで安全に受信・適用し、Draftグラフのリアルタイム更新を実現する。

## 対象

* `useActStream`（購読制御）
* `applyPatch`（state適用）
* Graph描画更新
* Thinkthrough表示（thought stream）

## 前提仕様

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/quality/frontend-stream-acceptance.md`
* `act/specs/behavior/act-flow.md`

## 適用ルール（MUST）

* `upsert`: block 追加/更新
* `append_md`: `contentMd` 追記（置換禁止）
* `request_id`（UUID）をリクエストごとに付与する
* 同一論理リクエストの再送時は同じ `request_id` を使う
* `error` 受信時: loading解除 + `error.stage/retryable` を表示へ反映
* `done` 受信時: loading解除
* `done/error` 二重終端でもUIが壊れない
* `llm_config` / `grounding_config` は送信時に明示設定できる
* `thinking_config.include_thoughts` を送信時に指定可能
* `stream_parts[].thought=true` を通常回答と分離表示する

## 状態管理

* stream state: `idle | running | error`
* request単位の識別子を持ち、古いstream更新を無視できる設計
* 部分結果は失敗時も保持
* thought buffer と answer buffer を別Stateで保持

## UX要件

* テキスト追記を視認できる更新頻度で描画
* 長文追記時にUI固まりを避ける（バッチ描画またはthrottle）
* エラー時に再実行導線を残す
* モード切替（Flash / Deep Research）と Web Grounding ON/OFF を操作可能にする
* Thinkthrough表示ON/OFFを切替可能にする

## 完了条件（DoD）

* 正常系で逐次描画し `done` 終端
* 異常系で `error` 終端し復帰可能
* 重複イベント受信でも状態破綻しない
* thought/answer が視覚的に分離される
