# Usecase: Join Workspace By Invite URL

## 目的

招待URLから workspace へ参加し、該当 workspace の tree にアクセスできるようにする。

## トリガー入力

* ユーザーが招待URLを開く

## 前提条件

* Google アカウントでログイン済み
* 招待URLトークンが有効期限内

参照:

* `act/act-api/specs/security/access-control-middleware.md`
* `act/act-api/specs/workspace/invite-link-api-contract.md`
* `act/specs/overview/act-architecture.md`

## 正常フロー

1. 招待URLトークンを検証
2. `workspaceId` への membership を付与
3. 現在の workspace を招待先へ切替
4. `RunAct` が `PERMISSION_DENIED` にならず実行できる

## 異常フロー

* トークン期限切れ: 参加不可、再招待を案内
* トークン不正: 参加不可

## 受け入れ条件（UI / Backend）

* 招待URL経由で初回参加が成功する
* 期限切れURLを再利用できない

## テスト対応（E2EケースID）

* `UC-WORKSPACE-INVITE-01` -> `E2E-WORKSPACE-INVITE-URL`
