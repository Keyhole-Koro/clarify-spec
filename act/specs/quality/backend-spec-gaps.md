# Backend Spec Gaps (Resolved Decisions)

この文書は、Actバックエンドの未確定項目に対する「決定値」を記録する。

## 1. 認証情報の正本ルール

決定:

* Firebase ID Token claim を正本とする
* `RunActRequest.uid` は互換用（deprecated）とし、認証正本に使わない
* 実処理で使う `uid` は常に token claim から採用する
* `RunActRequest.uid` が存在し claim と不一致の場合は `UNAUTHENTICATED`
* `workspace_id` は境界照合用途のみ

## 1.1 冪等キー（request_id）

決定:

* `RunActRequest.request_id` を必須化
* 形式は UUID（v4推奨）
* 冪等キーは `(token_uid, workspace_id, request_id)`
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

## 11. 招待リンク共有機能（解決済み）

決定:

* API契約を `act/act-api/specs/workspace/invite-link-api-contract.md` に固定
* usecase連携を `act/specs/usecases/workspace/join-workspace-by-invite-url.md` に固定
* 認可境界は `act/act-api/specs/security/access-control-middleware.md` を正本とする

## 12. Context Assembly 実装粒度（解決済み）

決定:

* 実装固定値は `context/assembly/implementation.md` を正本とする
* ADK Worker 実行プロファイルは `act/act-adk-worker/specs/context/context-assembly-execution-profile.md` を正本とする
* Firestore query/index 固定値を同仕様へ明記した

## 13. ErrorInfo 最終マッピング（解決済み）

決定:

* `code/stage/retryable/retry_after_ms` の最終表は `act/act-api/specs/reliability/error-mapping-matrix.md` を正本とする

## 14. 追加で固定した運用仕様（解決済み）

決定:

* idempotency: `act/act-api/specs/reliability/idempotency-policy.md`
* redis keyspace: `act/act-api/specs/platform/redis-keyspace-policy.md`
* cookie/cors: `act/act-api/specs/security/cookie-cors-policy.md`
* service-to-service auth: `act/act-api/specs/security/internal-service-auth.md`

## 15. ADK Worker 実装詳細ギャップ（未解決）

現状:

* `act-adk-worker` の runtime は主要フローを定義済みだが、実装時に差が出やすい境界が残っている

次アクション（要決定）:

* cancel伝播仕様
  * `act-api` からの中断を `GenerateWithModel` / 外部tool呼び出しへ即時伝播する方式
* stream順序保証
  * thought / answer / patch の flush順序と同一chunk内の並びルール
* model profile切替表
  * Gemini 3 Flash / grounding / deep research の選択条件を固定
* tool/grounding metadata 正規化
  * `RunActEvent` へ何を残し、何を圧縮/破棄するかを固定
* ADK workerテスト仕様
  * 最小ゴールデンケース（intent, budget超過, error mapping, stream順序）を追加

## 16. Frontend エラー可視化ポリシー（未解決）

現状:

* UI向けの `ErrorInfo.message` / `stage` / `retryable` の表示要件はあるが、開発時に dev console へ何を出してよいかの方針が未固定

次アクション（要決定）:

* dev console 出力対象
  * フロントが受信した `terminal.error` / stream例外 / reducer適用失敗 / 想定外eventを、表示可能な情報は dev console へ出す
* 出力粒度
  * `trace_id`, `request_id`, `stage`, `retryable`, `message`, raw event payload のうち、秘匿情報を除いてどこまで常時出すかを固定
* 環境境界
  * development 限定にするか、staging/preview でも有効にするかを固定
* PII/secret マスキング
  * token, cookie, authorization header, 個人情報を console 出力から除外するルールを固定

## 17. Frontend 実装前に詰めるべき仕様境界（未解決）

現状:

* 共有仕様の大枠は固定されているが、文書間で解釈差が残る箇所と、UI実装時に判断が分かれやすい境界がある

次アクション（要決定）:

* terminal 挙動の整合
  * `done/error` をサーバ契約として厳密排他にする一方、フロントは二重終端受信時に無害化する実装を許容するかを固定
* Deep Research fallback の UI 通知
  * fallback 発生をユーザーに表示するか、dev console/log のみで追跡するかを固定
* thought 表示 UX
  * thought 表示の初期値、実行中のみ表示するか、完了後も保持するか、再実行時のクリア条件を固定
* grounding / tool metadata の UI 投影
  * 参照リンク、根拠、diagnostics をどの UI 領域に出すか、未対応時は store 保持のみにするかを固定
