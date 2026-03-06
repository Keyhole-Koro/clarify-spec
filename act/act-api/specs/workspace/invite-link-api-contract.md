# Invite Link API Contract

## 目的

workspace 招待リンクの発行・検証・参加をバックエンド契約として固定する。

## スコープ / 非スコープ

* スコープ: API I/O、トークン寿命、失効条件、エラーコード
* 非スコープ: フロントUI配置、メール送信機能

## 前提・依存

* `act/specs/usecases/workspace/join-workspace-by-invite-url.md`
* `act/act-api/specs/security/access-control-middleware.md`
* `act/specs/quality/backend-parameter-index.md`

## 契約（I/O）

### 1) `POST /api/workspaces/{workspace_id}/invites`

入力:

* auth user（workspace member）
* optional `expires_in_seconds`

出力:

* `invite_url`
* `invite_token_id`
* `expires_at`

### 2) `POST /api/invites/validate`

入力:

* `invite_token`

出力:

* `valid`
* `workspace_id`
* `workspace_name`
* `expires_at`

### 3) `POST /api/invites/join`

入力:

* auth user
* `invite_token`

出力:

* `workspace_id`
* `membership_status=joined`

## 正常フロー

1. 発行時に署名付きトークンを生成
2. Firestore `invites/{inviteTokenId}` を `active` で保存
3. validate で署名・期限・status を確認
4. join で membership を upsert
5. 使い切りポリシー時は `used` へ遷移

## 異常フロー（error/retryable/stage）

* token不正: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* token期限切れ: `FAILED_PRECONDITION`, `retryable=false`, `stage=VALIDATE_REQUEST`
* token失効済み: `FAILED_PRECONDITION`, `retryable=false`, `stage=VALIDATE_REQUEST`
* 既存member: `ALREADY_EXISTS`, `retryable=false`, `stage=VALIDATE_REQUEST`
* 保存失敗: `UNAVAILABLE`, `retryable=true`, `stage=EMIT_STREAM`

## 数値パラメータ

* `invite_token_ttl_seconds_default=172800`（48h）
* `invite_token_ttl_seconds_max=604800`（7d）
* `invite_token_single_use=true`

## 受け入れ条件（DoD）

* validate/join で期限切れと失効を区別できる
* 招待URLから workspace 参加が再現できる
* トークン再利用時に正しく拒否できる
