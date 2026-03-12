# Frontend Agent Tool MCP Lifecycle

## 目的

frontend agent tools を MCP server として扱う際の lifecycle を固定し、session 開始から tool invocation, error, reconnect までの挙動を揃える。

## 対象

* MCP session 開始
* tool discovery
* tool invocation
* transport error / reconnect

## 前提 / 参照

* `act/specs/contracts/frontend-agent-tools-mcp.md`
* `act/specs/contracts/frontend-agent-tools.md`
* `act/specs/behavior/frontend-agent-tool-interactions.md`

## 基本フロー

1. agent が frontend MCP server へ接続する
2. MCP server が tool capability を公開する
3. agent が tool list を discovery する
4. agent が必要な tool を invoke する
5. frontend が tool result または tool error を返す
6. agent が必要なら `RunAct` を呼び、Context Assembly は backend 側で行う

## Discovery

* session 開始後、agent は tool list を取得できる
* tool list は `act/specs/contracts/frontend-agent-tools.md` と整合している必要がある
* frontend が未実装の optional tool は公開しない

## Invocation

* read tool は state を変更しない
* write tool は UI state を変更しても知識正本は更新しない
* validation failure は tool error を返す
* transport/session failure は MCP lifecycle 側のエラーとして扱う

## Reconnect

* session 切断後は reconnect 時に再 discovery できるようにする
* reconnect 後も tool contract version を確認できるようにする
* 一時 UI state の継続は best-effort とし、正本保証に使わない

## Error Handling

* MCP transport error は session/lifecycle 問題として扱う
* tool error は tool invocation の結果として扱う
* `report_stream_error` は frontend 内の stream 可観測性用であり、MCP transport error の代替ではない

## MUST / 制約

* MCP lifecycle は tool を agent に届けるための層であり、Context Assembly を実行しない
* `RunAct` 実行前の UI 操作と、`RunAct` 実行後の stream handling は責務を分離する
* reconnect 可否に依存して知識正本の整合性を保とうとしてはならない

## 完了条件（DoD）

* session 開始から discovery / invoke / reconnect までの挙動が説明できる
* tool error と transport error が区別されている
* MCP を導入しても tool と assembly の分離原則が崩れない
