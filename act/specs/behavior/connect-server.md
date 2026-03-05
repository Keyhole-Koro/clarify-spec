# Connect サーバ公開仕様

## 目的

`ActService.RunAct` を Connect RPC として公開し、フロント実装がそのまま接続できる運用境界を定義する。

## 公開対象

* Service: `ActService`
* RPC: `RunAct`（server-streaming）
* 実装スタック: Go + connect-go

## 前提仕様

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/runact-implementation.md`

## サーバ要件（MUST）

* streaming応答を途中切断なく返せる
* deadline/timeout超過時に明示エラーを返す
* requestごとに traceId を付与
* workspace/uid の認可チェックを行う
* Vertex AI 認証情報を解決し、Gemini呼び出し可能であること
* `thinking_config` / `research_config` を解釈できること

## 環境境界

* `RPC_BASE_URL` でendpoint切替
* dev/prod で公開ポリシーを分離
* CORSはfrontend originを明示許可
* `VERTEX_PROJECT_ID` / `VERTEX_LOCATION` を環境ごとに固定
* `GOOGLE_APPLICATION_CREDENTIALS` もしくは Workload Identity を使用

## セキュリティ

* 認証情報（uid/workspace）を必ず検証
* workspace越境は `FAILED_PRECONDITION` か `PERMISSION_DENIED` 相当で拒否
* ログへ機微情報本文を直接出さない

## 障害時挙動

* upstream障害時は `UNAVAILABLE` / `DEADLINE_EXCEEDED`
* 内部例外は `INTERNAL`
* 失敗ログに `traceId` を必須出力

## 完了条件（DoD）

* フロントから `RunAct` stream接続が成立
* 認可失敗時の拒否が確認できる
* traceIdでサーバログ追跡が可能
* Goサーバで stream 中断時のリソースリークがない
