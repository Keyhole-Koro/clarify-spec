# Connect RPC 利用方針

## 1. 目的

Frontend / Backend 間のRPC通信は Connect RPC を使用する。
スキーマは `rpc/specs/proto-contracts.md` を唯一の正本とする。

## 2. 生成

`rpc` ディレクトリで実行:

```bash
# Markdownのproto契約をもとに、必要時に一時的な .proto / buf設定を生成してから実行
buf generate
```

生成先:

* `gen/ts`（frontend用）
* `gen/go`（backend用）

## 3. Frontend（TypeScript）

主要ライブラリ:

* `@connectrpc/connect`
* `@connectrpc/connect-web`

方針:

* `createConnectTransport` を使い `NEXT_PUBLIC_RPC_BASE_URL` へ接続
* `ActService.RunAct` は server-streaming で受信
* 受信 `PatchOp` は順次 store の `applyPatch` に適用

## 4. Backend（Go/TSどちらでも可）

方針:

* `ActService` と `OrganizeService` を Connect サーバとして公開
* `RunAct` は read-only（Firestore直接更新なし）
* 永続化は `OrganizeService.ApplyPatch` のみ
* `RunAct` の内部オーケストレーションは LangGraph を推奨（`act/backend/act-langgraph-spec.md`）

## 5. 参照

* `rpc/specs/proto-contracts.md`
* `rpc/specs/buf-config.md`
* `act/backend/act-langgraph-spec.md`
* `act/backend/act-e2e-test-plan.md`
