# Usecase: RunAct Without Workspace Access

## 目的

workspace 未所属ユーザーによる `RunAct` を拒否し、越境アクセスを防ぐ。

## トリガー入力

* 所属していない `workspace_id` を指定して `RunAct` 送信

## 前提条件

* ユーザーは有効な Google token を持つ

参照:

* `act/act-api/specs/access-control-middleware.md`
* `act/act-api/specs/connect-server.md`

## 正常フロー

1. middleware が membership を検証
2. 未所属を検知
3. `PERMISSION_DENIED` で終了

## 異常フロー

* `tree_id` 不存在時は `FAILED_PRECONDITION`
* token 不正時は `UNAUTHENTICATED`

## 受け入れ条件（UI / Backend）

* handler に到達する前に拒否される
* 拒否ログに `traceId` と `workspace_id` が残る

## テスト対応（E2EケースID）

* `UC-WORKSPACE-AUTHZ-01` -> `E2E-WORKSPACE-PERMISSION-DENIED`
