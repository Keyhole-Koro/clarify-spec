# Frontend Agent Tools MCP Contract

## 目的

frontend agent tools を MCP 経由で agent に公開する際の discovery, invocation, session 境界を固定する。

## スコープ / 非スコープ

* スコープ: frontend tools の MCP 公開面
* 非スコープ: tool 自体の意味論、Context Assembly、本体 UI 実装

## 前提 / 参照

* `act/specs/contracts/frontend-agent-tools.md`
* `act/specs/behavior/frontend-agent-tool-interactions.md`
* `concept.md`

## 位置づけ

* MCP は frontend tools の公開方式である
* MCP は tool contract を agent に配る transport / session / discovery 層である
* MCP は Context Assembly を置き換えない
* tool contract = 中身、MCP = 配り方、assembly = 使った後の文脈化

## MCP Server Surface

### Server Identity

| field | 必須 | 説明 |
| --- | --- | --- |
| `server_name` | 必須 | frontend tool server 名 |
| `server_version` | 必須 | contract 互換性確認用 version |
| `capabilities.tools` | 必須 | tool invocation 対応 |

### Discovery

* MCP server は `act/specs/contracts/frontend-agent-tools.md` に定義した tool 一覧を公開する
* 各 tool は `name`, `description`, `input_schema` を含む
* frontend 固有 UI 文脈に依存する optional tool は capability として明示できる

### Invocation

* agent は MCP の tool invocation を通じて frontend tools を呼び出す
* 入力 validation は tool schema に従って frontend 側で行う
* validation failure は MCP error ではなく tool error として返せるようにする
* tool result は `output_schema` に従う

## Session Rules

* MCP session は frontend UI state の観測・操作のために使う
* session が切れても知識正本や Context Assembly の正本は変わらない
* reconnect 時は tool contract を再 discovery できるようにする
* selection group など一時 UI state の復元可否は frontend 実装ポリシーに従う

## Error Handling

* transport/session failure は MCP 層のエラーとして扱う
* `INVALID_INPUT`, `NOT_FOUND`, `CONFLICT`, `UNAVAILABLE`, `INTERNAL` は tool error として返す
* `report_stream_error` は MCP transport error ではなく、frontend 内の stream/error 可観測性のための tool である

## MUST / 制約

* MCP は tool invocation の境界であり、retrieval / ranking / budgeting を実行しない
* agent は tool result をそのまま model prompt に直結せず、必要な request input に変換して `RunAct` へ渡す
* Context Assembly は引き続き backend / worker 側で行う
* MCP session の存在を前提に frontend state を知識正本として扱ってはならない

## 完了条件（DoD）

* MCP の役割が tool contract と Context Assembly から分離して説明できる
* frontend tools を MCP discovery / invocation で公開できる
* transport error と tool error の責務が分かれている
