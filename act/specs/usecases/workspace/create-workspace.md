# Usecase: Create Workspace

## 目的

ユーザーが新しい workspace を作成し、以後その workspace で Act を実行できる状態にする。

## トリガー入力

* ユーザーが「新しい workspace 作成」を実行

## 前提条件

* Google アカウントでログイン済み

参照:

* `act/act-api/specs/access-control-middleware.md`
* `act/specs/overview/act-architecture.md`

## 正常フロー

1. workspace を新規作成
2. 作成者を `owner/member` として登録
3. 現在の workspace を新規IDへ切替
4. `RunAct` が当該 workspace で実行可能

## 異常フロー

* 作成失敗時は workspace を切替しない
* 権限付与失敗時は作成をロールバック

## 受け入れ条件（UI / Backend）

* 同一ユーザーが複数 workspace を作成できる
* 作成直後に認可エラーなく `RunAct` を呼べる

## テスト対応（E2EケースID）

* `UC-WORKSPACE-CREATE-01` -> `E2E-WORKSPACE-CREATE`
