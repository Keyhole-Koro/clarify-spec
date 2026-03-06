# Cookie and CORS Policy（Act API）

## 目的

Cookie運用とCORS許可条件を固定し、環境差分での認証失敗を防ぐ。

## スコープ / 非スコープ

* スコープ: sid/csrf cookie属性、origin許可、credentials運用
* 非スコープ: CDN/WAF詳細

## 前提・依存

* `act/act-api/specs/security/session-and-auth-boundary.md`
* `act/act-api/specs/runtime/connect-server.md`
* `frontend/frontend-spec.md`

## 契約（I/O）

入力:

* Cookie `sid`, `csrf_token`
* Header `Origin`, `X-CSRF-Token`, `Authorization`

出力:

* 許可originのみ `Access-Control-Allow-Origin` を返却
* 認証成功時に cookie 更新

## Cookie 属性（固定）

* `sid`:
  * `HttpOnly=true`
  * `Secure=true`
  * `SameSite=Lax`
  * `Path=/`
  * `Max-Age=86400`
* `csrf_token`:
  * `HttpOnly=false`
  * `Secure=true`
  * `SameSite=Lax`
  * `Path=/`
  * `Max-Age=86400`

## CORS 固定ルール

* `Access-Control-Allow-Credentials=true`
* `Allow-Origin` は whitelist のみ
* `*` は禁止
* 許可ヘッダ: `Authorization`, `Content-Type`, `X-CSRF-Token`
* 許可メソッド: `POST`, `OPTIONS`

## Environment別 Allow-Origin（固定）

* dev: `http://localhost:3000`
* staging: `https://stg.<your-domain>`
* prod: `https://app.<your-domain>`

## 異常フロー（error/retryable/stage）

* origin不許可: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* csrf不一致: `PERMISSION_DENIED`, `retryable=false`, `stage=CSRF_VALIDATE`
* cookie欠落（soft）: 継続 + degradeログ
* cookie欠落（strict）: `UNAUTHENTICATED`, `retryable=true`, `stage=SID_VALIDATE`

## 数値パラメータ

* `sid_ttl_seconds=86400`
* `csrf_ttl_seconds=86400`
* `cors_preflight_max_age_seconds=600`

## 受け入れ条件（DoD）

* frontend が `credentials: include` で安定接続できる
* 本番で `*` CORS が使われない
* sid を JS から参照できない
