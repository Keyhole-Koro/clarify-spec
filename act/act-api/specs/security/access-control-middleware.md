# Access Control Middleware 仕様

## 目的

認証・認可の境界を共通 middleware に集約し、`tree_id` と `workspace_id` の不整合を防ぐ。

## スコープ / 非スコープ

* スコープ: Act API の request 前段で行う検証
* 非スコープ: セッション保持実装、UI上の招待導線詳細

## 前提 / 参照

* `act/act-api/specs/runtime/connect-server.md`
* `act/act-api/specs/security/session-and-auth-boundary.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/overview/act-architecture.md`

## 契約（I/O）

入力:

* Firebase ID Token（`Authorization`）
* `workspace_id`, `tree_id`, `uid`（request）

出力:

* 検証通過時のみ handler（`RunAct`）へ委譲
* 失敗時は `UNAUTHENTICATED` / `PERMISSION_DENIED` / `FAILED_PRECONDITION`

## 検証チェーン（MUST）

1. Firebase ID Token 検証
2. `firebase.sign_in_provider == google.com` を確認
3. token `uid` と request `uid` を照合
4. token user が request `workspace_id` メンバーであることを確認
5. request `tree_id` が request `workspace_id` に属することを確認
6. すべて通過時のみ handler (`RunAct`) へ委譲

補足:

* `sid` は認可判定の正本に使わない
* 認可正本は常に `token user -> workspace membership -> tree access`

## データモデル前提（最小）

* `workspaces/{workspaceId}`
* `workspaces/{workspaceId}/members/{uid}`
* `trees/{treeId}` に `workspace_id` を保持

## Workspace運用要件

* ユーザーは workspace を複数作成できる（上限は当面設けない）
* 招待URLで workspace 参加できる
* 招待URLは期限付きトークン（短寿命）を使用する

## エラーコード方針

* token不正 / provider不一致 / uid不一致: `UNAUTHENTICATED`
* workspace未所属 / tree越境: `PERMISSION_DENIED`
* tree不存在: `FAILED_PRECONDITION`

## 数値パラメータ

* 追加の独自パラメータは持たない
* timeout/retry は上位仕様（`connect-server.md`, `backend-parameter-index.md`）を使用する

## 完了条件（DoD）

* `RunAct` 実行前に検証チェーンが必ず通る
* middleware の通過/拒否がログ（traceId）で追える
