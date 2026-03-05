# Frontend Stream 統合仕様

## 目的

`RunActEvent` をフロントで安全に受信・適用し、Draftグラフのリアルタイム更新を実現する。

## 対象

* `useActStream`（購読制御）
* `applyPatch`（state適用）
* Graph描画更新

## 前提仕様

* `act/specs/rpc-connect-schema.md`
* `act/frontend/act-stream-acceptance.md`
* `act/specs/act-flow.md`

## 適用ルール（MUST）

* `upsert`: block 追加/更新
* `append_md`: `contentMd` 追記（置換禁止）
* `error` 受信時: loading解除 + エラー表示
* `done` 受信時: loading解除
* `done/error` 二重終端でもUIが壊れない

## 状態管理

* stream state: `idle | running | error`
* request単位の識別子を持ち、古いstream更新を無視できる設計
* 部分結果は失敗時も保持

## UX要件

* テキスト追記を視認できる更新頻度で描画
* 長文追記時にUI固まりを避ける（バッチ描画またはthrottle）
* エラー時に再実行導線を残す

## 完了条件（DoD）

* 正常系で逐次描画し `done` 終端
* 異常系で `error` 終端し復帰可能
* 重複イベント受信でも状態破綻しない
