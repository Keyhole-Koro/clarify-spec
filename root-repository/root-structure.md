# ルートリポジトリ構成案（Proto集約 + Submodule運用）

前提:

* ルート repo に proto 契約仕様を置く
* frontend/backend はサブモジュールで管理する
* 生成はルートで 1 回実行する

## 推奨ディレクトリ構成（自然言語）

### ルート直下

* `README.md`, `LICENSE`, `.gitignore`, `.editorconfig`

### `docs/`

* 他チーム連携用の仕様・図
* 例: `docs/specs/act-ui-patch-spec-v0.1.md`, `docs/specs/rpc-contract.md`, `docs/specs/firestore-schema.md`
* 例: `docs/diagrams/system-overview.png`, `docs/diagrams/block-tree.png`

### `rpc/`

* `rpc/specs/proto-contracts.md`（共有RPC契約）
* `rpc/specs/buf-config.md`（buf設定仕様）

### `gen/`

* 生成物の配布元
* `gen/ts/connect`, `gen/go/connect`, `gen/README.md`

### `tools/`

* 生成・整形・CI補助スクリプト
* 例: `tools/scripts/gen_proto.sh`, `tools/scripts/sync_gen_to_submodules.sh`

### `apps/`

* サブモジュール置き場
* `apps/frontend`, `apps/backend`

### その他

* `.github/workflows/ci.yml`（任意）
* `docker/compose.yaml`（任意）

## RPC契約の切り方

### `common/v1` に置くべき要素

* UI Composer の Block / Action / PatchOp
* UiCapabilities / UiPreferences / Anchor / HistoryItem
* Act enum（EXPLORE/CONSULT/INVESTIGATE）
* StreamEvent（patch/error/done）

### `act/v1` に置くべき要素

* `RunAct` の request/stream response 契約
* Act UI Patch Spec v0.1 の protobuf相当仕様

### `organize/v1` に置くべき要素

* MVPで未使用でも将来拡張用に置き場を確保

## サブモジュール運用ポイント

* RPC契約の正本は `rpc/specs/proto-contracts.md`
* 生成設定の正本は `rpc/specs/buf-config.md`
* 生成物は `gen/` へ集約し、同期スクリプトで frontend/backend へ配布する
