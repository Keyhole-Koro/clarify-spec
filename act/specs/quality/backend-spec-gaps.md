# Backend Spec Gaps (Resolved Decisions)

この文書は、Actバックエンドの未確定項目に対する「決定値」を記録する。

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

## 2. 認可境界（workspace/tree/topic）

決定:

* `topic.workspace_id == request.workspace_id` を必須化
* `tree_id` 指定時は `tree.workspace_id == request.workspace_id` を必須化
* `token.uid` が `members/{workspace_id}` に存在することを必須化
* 上記違反時は `PERMISSION_DENIED`
* 認可は共通 middleware に集約する

## 3. ADK導入境界（新規決定）

決定:

* Go `act-api` は契約ゲートウェイとして維持
* ADKは別Cloud Run `act-adk-worker` で実行
* `RunAct` 契約（Connect RPC / PatchOp）は変更しない
* ADK workerは read-only（Firestore/GCS write禁止）

## 3.1 ADK失敗時挙動

決定:

* worker到達不可: `UNAVAILABLE`
* worker timeout: `DEADLINE_EXCEEDED`
* 返却形式不正: `INTERNAL`
* すべて `ErrorInfo.stage` を埋めて返す

## 4. ストリーム切断時挙動

決定:

* client切断検知時に context cancel を必須化
* 再接続時は再開しない（再実行のみ）

## 5. レート制限 / 同時実行制御

決定:

* 同時Run上限: `uid <= 4`, `workspace <= 60`
* 超過時コード: `RESOURCE_EXHAUSTED`
* `Retry-After`: 3秒

## 6. Deep Research運用境界

決定:

* polling interval: 3秒
* max wait: 45秒
* fallback条件: timeout または 5xx 2連続

## 7. エラーコードマッピング

決定:

* 外部/内部例外を `ErrorInfo.code` へ正規化
* 詳細内部情報はレスポンスへ出さずログへ出す
* `ErrorInfo.retryable` と `ErrorInfo.stage` を必須化

## 8. Cloud Runデプロイ固定値

決定:

* `act-api`: `concurrency=40`, `timeout=120s`, `memory=2Gi`, `min=0`, `max=20`
* `act-adk-worker`: `concurrency=10`, `timeout=120s`, `memory=2Gi`, `min=0`, `max=20`

## 9. ログ/メトリクス粒度

決定:

* `load_context_ms`
* `adk_worker_latency_ms`
* `model_first_token_ms`
* `stream_duration_ms`
* `adk_worker_error_total`

## 10. 数値設定の正本

* `act/specs/quality/backend-parameter-index.md`

## 11. 招待リンク共有機能（未実装ギャップ）

現状:

* 共有リンク発行/検証のバックエンド契約が未定義
* フロント共有UIが未定義

次アクション:

* backend仕様: `act/act-api/specs/access-control-middleware.md` と `act/specs/usecases/workspace/join-workspace-by-invite-url.md` にAPI契約追記
* frontend仕様: `frontend/frontend-spec.md` に共有/参加UIの最小ワイヤー追記

## 12. Context Assembly 実装粒度ギャップ（未固定）

現状:

* `ResolveIntent` / `RetrieveContext` / `RankAndBudget` は概念定義が中心
* 実装時に揺れやすい定数・優先順位が未固定

次アクション（固定必須）:

* `intent` 判定の優先順位表（競合時ルール）
* `retrieval_policy` 固定値（hop、relation、件数上限）
* `ranking` 重み定数（score計算）
* `token budgeting` 推定式と削除順序
* Firestore query条件 + 必要index一覧
* `ErrorInfo` 最終マッピング（`code/stage/retryable/retry_after_ms`）
