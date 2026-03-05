# Firestore Security Rules (Policy Draft)

`google.com` 認証 + workspace境界を前提にしたルール方針。

## MUST

* 未認証アクセスを拒否
* `request.auth.token.firebase.sign_in_provider == \"google.com\"` を要求
* `workspaces/{workspaceId}/members/{uid}` 存在確認を必須化
* `tree.workspace_id == workspaceId` の整合を要求

## Read / Write Boundary

* Read:
  * workspace member のみ許可
* Write:
  * `owner` または要件に応じた `member` に限定
* Invite:
  * 招待トークンは期限チェック必須

## Notes

* 詳細ルール実装時は `firestore/schema.md` の制約と一致させる
