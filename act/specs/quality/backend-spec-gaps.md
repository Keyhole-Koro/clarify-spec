# Backend Spec Gaps (Act)

この文書は、Actバックエンド仕様で未確定な項目を実装前に固定するためのギャップ一覧である。
対象は Cloud Run + Connect Go + Firebase Auth + Firestore + Vertex AI。

## 1. 認証情報の正本ルール不足

不足:

* `RunActRequest.uid` / `workspace_id` と Firebase ID Token claim の優先順位が未固定

決めること:

* token claim を正本とし、request値は照合用途のみとするか
* 不一致時のエラーコード（`UNAUTHENTICATED` or `PERMISSION_DENIED`）

## 2. 認可境界（workspace/tree）不足

不足:

* `tree_id` が `workspace_id` に属さない場合の判定手順が曖昧
* Firestore読取の許可条件が未定義

決めること:

* `tree.workspace_id == token.workspace_id` の必須化
* 越境時コードを `PERMISSION_DENIED` へ固定

## 3. ストリーム切断時挙動不足

不足:

* client切断時にLLM呼び出しを即cancelする規約が未定義
* 再接続時に「再開」か「再実行」かが未定義

決めること:

* 切断検知で context cancel を必須化
* 再接続は再開不可、再実行のみ許可とするか

## 4. レート制限 / 同時実行制御不足

不足:

* `uid` / `workspace` 単位の同時Run上限が未定義
* 429返却条件とRetry-After方針が未定義

決めること:

* 例: `uidあたり同時2`, `workspaceあたり同時20`
* 超過時 `UNAVAILABLE` ではなく `RESOURCE_EXHAUSTED` 相当へ固定

## 5. Deep Research運用境界不足

不足:

* Interactions polling間隔・最大待機時間・中断条件が未定義
* フォールバック閾値（何秒でFlashへ切替）が曖昧

決めること:

* polling interval（例: 2s）
* max wait（例: 25s）
* fallback条件（timeout/5xx連続回数）

## 6. エラーコードマッピング表不足

不足:

* Vertex/Firebase/Firestore/内部例外のマッピングが一枚表になっていない

決めること:

* 例外種別 -> `ErrorInfo.code` の対応表を契約化
* ユーザー表示用メッセージとの分離（内部詳細を漏らさない）

## 7. Cloud Runデプロイ固定値不足

不足:

* `concurrency`, `timeout`, `memory`, `min instances`, `max instances` が未固定
* service account IAM の最小権限セットが未固定

決めること:

* ハッカソン初期推奨値（例: concurrency 20, timeout 60s, min 0）
* 必要ロール（Firestore read, Vertex invoke, Logging write）

## 8. ログ/メトリクス粒度不足

不足:

* stage別レイテンシの分解指標が不足
* cutoff（どの閾値で警告するか）が未定義

決めること:

* `load_context_ms`, `model_first_token_ms`, `stream_duration_ms` 追加
* p95/p99閾値の運用基準

## 9. 負荷・耐障害試験不足

不足:

* 同時接続、長時間stream、切断復帰の試験仕様が不足

決めること:

* 最低負荷ケース（同時N接続、3分stream）
* failover試験（Vertex 503連続）と期待挙動

## 優先度（実装前に決める順）

1. 認証/認可正本（1,2）
2. ストリーム切断・再実行（3）
3. レート制限（4）
4. Deep Research境界（5）
5. エラー表（6）
6. Cloud Run固定値（7）
7. 観測/負荷（8,9）
