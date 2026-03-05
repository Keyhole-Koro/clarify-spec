# Backend Spec Gaps (Resolved Decisions)

この文書は、Actバックエンドの未確定項目に対する「決定値」を記録する。
リミット関連はレビュー前提で緩めの設定にしている。

## 1. 認証情報の正本ルール

決定:

* Firebase ID Token claim を正本とする
* `RunActRequest.uid` / `workspace_id` は照合用途のみ
* claim と不一致は `UNAUTHENTICATED`

## 1.1 冪等キー（request_id）

決定:

* `RunActRequest.request_id` を必須化
* 形式は UUID（v4推奨）
* 冪等キーは `(uid, workspace_id, request_id)`
* 同一キーの重複実行は `ALREADY_EXISTS`

## 2. 認可境界（workspace/tree）

決定:

* `tree.workspace_id == token.workspace_id` を必須化
* 越境時は `PERMISSION_DENIED`
* 認可は共通 middleware に集約し、各handlerで重複実装しない

## 2.1 Workspace作成・招待URL

決定:

* ユーザーは workspace を複数作成可能（当面上限なし）
* 招待URLで workspace 参加可能
* 招待URLは期限付きトークン（短寿命）を利用

## 3. ストリーム切断時挙動

決定:

* client切断検知時に context cancel を必須化
* 再接続時は再開しない（再実行のみ）

## 4. レート制限 / 同時実行制御（緩和済み）

決定:

* 同時Run上限: `uid <= 4`, `workspace <= 60`
* 超過時コード: `RESOURCE_EXHAUSTED`
* `Retry-After`: 3秒

## 5. Deep Research運用境界（緩和済み）

決定:

* polling interval: 3秒
* max wait: 45秒
* fallback条件: timeout または 5xx 2連続

## 6. エラーコードマッピング表

決定:

* 外部/内部例外を `ErrorInfo.code` へ正規化する
* 詳細内部情報はレスポンスへ出さず、ログへ出す
* `ErrorInfo.retryable` と `ErrorInfo.stage` を必須化する

## 7. Cloud Runデプロイ固定値（緩和済み）

決定:

* `concurrency=40`
* `timeout=120s`
* `memory=2Gi`
* `min-instances=0`, `max-instances=20`

## 8. ログ/メトリクス粒度

決定:

* `load_context_ms`
* `model_first_token_ms`
* `stream_duration_ms`
* 既存メトリクスに追加して計測

## 9. 負荷・耐障害試験

決定:

* 同時接続: 40
* 長時間stream: 5分
* failover: Vertex 503注入時の再試行と終端コードを検証

## 10. 数値設定の正本

リミット・タイムアウト等の細かい数値は以下を正本とする。

* `act/specs/quality/backend-parameter-index.md`
