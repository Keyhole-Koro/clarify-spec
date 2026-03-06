# Usecase: Deep Research Fallback

## 目的

Deep Research実行中の遅延・失敗時に、UIを壊さず通常Flashプロファイルへフォールバックする。

## トリガー入力

* `research_config.use_deep_research=true` で RunAct 実行

## 前提条件

* Vertex Interactions API の状態取得が可能
* タイムアウト/失敗時のフォールバック設定が有効

参照:

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/contracts/gemini-vertex-response-schemas.md`
* `act/act-api/specs/runtime/runact-implementation.md`

## 正常フロー

1. Deep Researchで実行開始
2. 進捗または部分結果を受信
3. 完了時に `done=true` 終端

## 異常フロー

1. timeout または `UNAVAILABLE` を検知
2. サーバ側で Flashプロファイルへ切替
3. 切替後streamを継続し、最終的に `done` または `error` で終端

## ストリーム期待値（RunActEvent）

* 失敗時も `done/error` 排他を維持
* フォールバック発生を `warnings` またはログで追跡可能
* 最低1回は `patch_ops` または `text_delta` が返る（完全沈黙を避ける）

## 受け入れ条件（UI / Backend）

* UIはタイムアウト時に処理中断と誤認しない
* フォールバック時も同一リクエストとして扱える
* Backendログで `traceId` 単位に切替経路が追える

## テスト対応（E2EケースID）

* `UC-DEEP-FALLBACK-01` -> `E2E-DEEPRESEARCH-FALLBACK`
