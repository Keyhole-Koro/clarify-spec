# Frontend Stream 統合仕様

## 目的

`RunActEvent` をフロントで安全に受信・適用し、Draftグラフのリアルタイム更新を実現する。

## 対象

* Stream Adapter（購読制御）
* Patch Reducer（state適用）
* Graph Projection（描画変換）
* Thinkthrough表示（thought stream）

## 前提仕様

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/quality/frontend-stream-acceptance.md`
* `act/specs/behavior/act-flow.md`
* `act/specs/behavior/ai-streaming-cases.md`

## 適用ルール（MUST）

* `upsert`: block 追加/更新
* `append_md`: `contentMd` 追記（置換禁止）
* node 本文 state の正本は `append_md` とし、`stream_parts` / `text_delta` は逐次表示用バッファとして扱う
* `request_id`（UUID）をリクエストごとに付与する
* 同一論理リクエストの再送時は同じ `request_id` を使う
* `error` 受信時: loading解除 + `error.stage/retryable` を表示へ反映
* `done` 受信時: loading解除
* サーバ契約は `done/error` 排他とする
* frontend は防御的に最初の terminal のみ採用し、後続の duplicate terminal は UI へ反映せず無害化する
* `llm_config` / `grounding_config` は送信時に明示設定できる
* `thinking_config.include_thoughts` を送信時に指定可能
* `stream_parts[].thought=true` を通常回答と分離表示する
* thought 表示の既定は OFF とする
* thought は request 単位の専用 buffer で保持し、node 本文へ混ぜない
* `append_md` の適用前に対象 block の `upsert` が存在する前提で reducer を実装する
* 同一 event 内では `thought -> answer -> upsert -> append_md -> metadata -> terminal` の順序を前提にできる

## 状態管理

* stream state: `idle | running | error`
* request単位の識別子を持ち、古いstream更新を無視できる設計
* 部分結果は失敗時も保持
* thought buffer と answer buffer を別Stateで保持
* thought buffer は request ごとに分離し、再実行時に前回 request の thought を継承しない
* terminal は request 単位で最初の1件だけを有効とする
* reducerは純粋関数として実装し、副作用を持たない
* graph projectionは読み取り専用でstate更新を行わない
* metadata は本文 state 更新と別Stateで保持できる設計にする

## UX要件

* テキスト追記を視認できる更新頻度で描画
* 長文追記時にUI固まりを避ける（バッチ描画またはthrottle）
* エラー時に再実行導線を残す
* モード切替（Flash / Deep Research）と Web Grounding ON/OFF を操作可能にする
* Thinkthrough表示ON/OFFを切替可能にする
* thought 有効時は実行中に展開表示でき、終端後は既定で折りたたむ
* Deep Research fallback 発生時は、エラー通知ではなく stream status 領域の軽いインライン通知で示す
* fallback 通知文言は「Deep Research から標準モードへ切り替えて継続中」であることを伝える
* fallback の詳細理由は UI より dev console / log を正本とする
* grounding metadata は本文下の `References` セクションへ投影する
* tool metadata は既定で `Diagnostics` 側へ寄せ、本文主表示へ混ぜない
* metadata は未対応項目を含めて store と dev console では追跡可能にする
* duplicate terminal 検知時は dev console に warning を残せるようにする

## 完了条件（DoD）

* 正常系で逐次描画し `done` 終端
* 異常系で `error` 終端し復帰可能
* 重複イベント受信でも状態破綻しない
* thought/answer が視覚的に分離される
