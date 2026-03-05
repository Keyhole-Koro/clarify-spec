# ルートリポジトリ構成案（Proto集約 + Submodule運用）

前提:

* ルート repo に proto を置く
* frontend/backend はサブモジュールで管理する
* 生成はルートで 1 回実行する

```txt
root/
  README.md
  LICENSE
  .gitignore
  .editorconfig

  docs/                         # 他チーム連携用の仕様・図
    specs/
      act-ui-patch-spec-v0.1.md # Act UI Patch Spec v0.1（プロンプト+スキーマ）の確定版
      rpc-contract.md           # act/organize RPCの契約（入力/出力、ストリーム仕様）
      firestore-schema.md       # nodes/trees/evidence のコレクション設計
    diagrams/
      system-overview.png       # 任意：全体図
      block-tree.png            # 任意：UI block tree図

  rpc/
    specs/
      proto-contracts.md        # ✅ 共有RPC契約（protoをMarkdownで管理）
      buf-config.md             # ✅ buf.yaml/buf.gen.yamlをMarkdownで管理

  gen/                          # ✅ ルートで生成物を集約（配布元）
    ts/
      connect/                  # Connect/TS用生成物（buf generateの出力先）
    go/
      connect/                  # Go用生成物（将来Goに振れてもOKなように）
    README.md                   # 生成方法・配布方法（submoduleへの反映ルール）

  tools/                        # 生成・整形・CI補助スクリプト
    scripts/
      gen_proto.sh              # buf generate を叩く（gen/ts, gen/go に出す）
      sync_gen_to_submodules.sh # gen/ts を frontend/src/gen に、gen/ts を backend/src/gen に配布
      lint.sh
      fmt.sh
    ci/
      check_generated.sh        # 生成物差分チェック（任意）

  apps/                         # ✅ サブモジュール置き場（git submodule）
    frontend/                   # submodule: SPA (Next.js)  ※このrepoの中身は別
    backend/                    # submodule: Cloud Run (TS) ※このrepoの中身は別

  .github/                      # 任意：GitHub Actionsで生成・チェック
    workflows/
      ci.yml

  docker/                       # 任意：ローカル開発補助
    compose.yaml                # emulator等を使うなら（Firestore emulatorなど）
```

---

## RPC契約の切り方（この構造でのおすすめ）

### `proto-contracts.md` 内 `common/v1/types.proto` 節

* UI Composer の **Block / Action / PatchOp**（v0.1: upsert/append_md）
* `UiCapabilities`, `UiPreferences`, `Anchor`, `HistoryItem`
* `Act` enum（EXPLORE/CONSULT/INVESTIGATE）
* `StreamEvent`（patch/error/done）を共通にしてもOK

### `proto-contracts.md` 内 `act/v1/act.proto` 節

* `service ActService { rpc RunAct(RunActRequest) returns (stream StreamEvent); }`
* Act UI Patch Spec v0.1 をそのまま Protobuf 化

### `proto-contracts.md` 内 `organize/v1/organize.proto` 節

* MVPでは不要なら空でもOK
* 後で「organize 書き込み側RPC」を足すときの置き場だけ確保

---

## “サブモジュール運用”のポイント（この構造の狙い）

* **RPC契約は `rpc/specs/proto-contracts.md` が唯一の正**
* 生成は `rpc/specs/buf-config.md` の設定から一時生成して実行
* 生成物は `root/gen/ts` に出して、`sync_gen_to_submodules.sh` で

  * `apps/frontend/src/gen/rpc` へ
  * `apps/backend/src/gen/rpc` へ
    **配布**する（ハッカソンでセットアップが壊れにくい）
