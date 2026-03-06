# Internal Service Auth（act-api -> act-adk-worker）

## 目的

`act-api` から `act-adk-worker` への内部呼び出しを OIDC で保護し、なりすまし呼び出しを防ぐ。

## スコープ / 非スコープ

* スコープ: 認証方式、Cloud Run IAM、token検証
* 非スコープ: mTLS、service mesh 実装

## 前提・依存

* `act/act-api/specs/runtime/adk-service-integration.md`
* `act/act-api/specs/runtime/connect-server.md`

## 契約（I/O）

入力:

* `act-api` service account による OIDC ID Token
* audience = `act-adk-worker` URL

出力:

* 認証成功時のみ worker 処理継続
* 失敗時 `UNAUTHENTICATED`

## 正常フロー

1. `act-api` が GCP metadata 経由で ID Token を取得
2. `Authorization: Bearer <id_token>` で worker を呼ぶ
3. `act-adk-worker` が audience と issuer を検証
4. `act-api` SA の `roles/run.invoker` を確認
5. 成功時のみ RunAct 内部処理へ進む

## 異常フロー（error/retryable/stage）

* token欠落: `UNAUTHENTICATED`, `retryable=false`, `stage=AUTHN`
* audience不一致: `UNAUTHENTICATED`, `retryable=false`, `stage=AUTHN`
* invoker権限なし: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* metadata取得失敗: `UNAVAILABLE`, `retryable=true`, `stage=AUTHN`

## 数値パラメータ

* `internal_auth_token_cache_ttl_seconds=300`
* `internal_auth_retry_max=1`

## 受け入れ条件（DoD）

* Public クライアントから worker 直接呼び出しできない
* `act-api` からのみ worker 呼び出しが成功する
* auth失敗時に `stage=AUTHN|AUTHZ` が一貫する
