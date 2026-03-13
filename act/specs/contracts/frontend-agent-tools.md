# Frontend Agent Tools Contract

## 目的

Act の agent が frontend の UI 能力を安全かつ再利用可能な `tool` として利用できるよう、tool 名、description、入出力 schema、失敗時挙動を固定する。

## スコープ / 非スコープ

* スコープ: frontend が agent に公開する read/write tool contract
* 非スコープ: Connect RPC 本体、React 実装詳細、LLM プロンプト本文

## 前提 / 参照

* `concept.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/frontend-stream-integration.md`
* `act/specs/behavior/frontend-canvas-phases.md`

## 設計原則

* tool は画面部品単位ではなく、ユーザー意図単位で設計する
* read tool と write tool を分離する
* frontend state は tool の入出力面であり、知識正本ではない
* write tool は冪等または冪等に近い再実行可能性を優先する
* 失敗時は agent が復帰判断できる最小エラー情報を返す

## 共通 Tool Envelope

### Tool Definition

| field | 必須 | 説明 |
| --- | --- | --- |
| `name` | 必須 | tool の一意名 |
| `description` | 必須 | agent に与える自然言語 description |
| `input_schema` | 必須 | JSON object schema |
| `output_schema` | 必須 | JSON object schema |

### Tool Error

| field | 必須 | 説明 |
| --- | --- | --- |
| `code` | 必須 | `INVALID_INPUT` / `NOT_FOUND` / `CONFLICT` / `UNAVAILABLE` / `INTERNAL` |
| `message` | 必須 | agent / dev console 向けメッセージ |
| `retryable` | 必須 | 再試行可否 |
| `details` | 任意 | 機械処理向けの追加情報 |

## 共通型

### NodeRef

| field | 必須 | 説明 |
| --- | --- | --- |
| `node_id` | 必須 | ノードID |
| `label` | 任意 | 表示ラベル |

### VisibleNode

| field | 必須 | 説明 |
| --- | --- | --- |
| `node_id` | 必須 | ノードID |
| `block_type` | 必須 | `ACT_ROOT` などの block 種別 |
| `title` | 任意 | ノード見出し |
| `content_md` | 任意 | 現在表示中の markdown |
| `parent_id` | 任意 | 構造親ノードID |
| `selected` | 必須 | 現在選択中か |
| `source` | 必須 | `frontend_draft` | `persisted` |
| `draft_revision` | 任意 | frontend draft revision |
| `persisted_revision` | 任意 | 永続化済み revision |
| `save_state` | 任意 | `streaming` | `completed_unsaved` | `persisted` | `failed` |
| `x` | 任意 | 描画上のX座標 |
| `y` | 任意 | 描画上のY座標 |

### VisibleEdge

| field | 必須 | 説明 |
| --- | --- | --- |
| `edge_id` | 必須 | エッジID |
| `source_node_id` | 必須 | 始点 |
| `target_node_id` | 必須 | 終点 |
| `edge_type` | 必須 | `tree` | `context` |

## Read Tools

### `get_visible_graph`

description:

* 現在 frontend に描画されているノード、エッジ、選択状態、active node を取得する

input_schema:

```json
{
  "type": "object",
  "properties": {
    "include_content": { "type": "boolean", "default": false },
    "selected_only": { "type": "boolean", "default": false }
  },
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "nodes": { "type": "array" },
    "edges": { "type": "array" },
    "selected_node_ids": { "type": "array" },
    "active_node_id": { "type": ["string", "null"] }
  },
  "required": ["nodes", "edges", "selected_node_ids", "active_node_id"],
  "additionalProperties": false
}
```

### `get_selected_nodes`

description:

* 現在選択中のノード集合を、agent が次の文脈入力に使える形で取得する

input_schema:

```json
{
  "type": "object",
  "properties": {
    "include_content": { "type": "boolean", "default": true }
  },
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "nodes": { "type": "array" },
    "count": { "type": "integer", "minimum": 0 }
  },
  "required": ["nodes", "count"],
  "additionalProperties": false
}
```

### `get_active_node_detail`

description:

* 右ペインで注目中のノード詳細を取得する。active node が未設定なら `null` を返す

input_schema:

```json
{
  "type": "object",
  "properties": {},
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "active_node_id": { "type": ["string", "null"] },
    "title": { "type": ["string", "null"] },
    "content_md": { "type": ["string", "null"] },
    "actions": { "type": "array" }
  },
  "required": ["active_node_id", "title", "content_md", "actions"],
  "additionalProperties": false
}
```

## Write Tools

### `select_nodes`

description:

* frontend 上の選択ノード集合を更新する。複数選択コンテキストの入力面として使う

input_schema:

```json
{
  "type": "object",
  "properties": {
    "node_ids": {
      "type": "array",
      "items": { "type": "string" }
    },
    "mode": {
      "type": "string",
      "enum": ["replace", "add", "remove"],
      "default": "replace"
    }
  },
  "required": ["node_ids"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "selected_node_ids": { "type": "array" },
    "count": { "type": "integer", "minimum": 0 }
  },
  "required": ["selected_node_ids", "count"],
  "additionalProperties": false
}
```

### `open_node_detail`

description:

* 指定ノードを active node に設定し、右ペイン詳細表示を開く

input_schema:

```json
{
  "type": "object",
  "properties": {
    "node_id": { "type": "string" }
  },
  "required": ["node_id"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "active_node_id": { "type": "string" },
    "opened": { "type": "boolean" }
  },
  "required": ["active_node_id", "opened"],
  "additionalProperties": false
}
```

### `create_selectable_nodes`

description:

* agent が候補ノード群を frontend 上に一時表示し、ユーザーに選択させるための選択用ノードセットを作成する

input_schema:

```json
{
  "type": "object",
  "properties": {
    "title": { "type": "string" },
    "instruction": { "type": "string" },
    "selection_mode": {
      "type": "string",
      "enum": ["single", "multiple"],
      "default": "single"
    },
    "options": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "properties": {
          "option_id": { "type": "string" },
          "label": { "type": "string" },
          "content_md": { "type": ["string", "null"] },
          "parent_id": { "type": ["string", "null"] },
          "metadata": { "type": ["object", "null"] }
        },
        "required": ["option_id", "label"],
        "additionalProperties": false
      }
    },
    "anchor_node_id": { "type": ["string", "null"] },
    "expires_in_ms": { "type": ["integer", "null"], "minimum": 1 }
  },
  "required": ["title", "instruction", "options"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "selection_group_id": { "type": "string" },
    "created_node_ids": { "type": "array" },
    "selection_mode": { "type": "string", "enum": ["single", "multiple"] },
    "pending_user_selection": { "type": "boolean" }
  },
  "required": ["selection_group_id", "created_node_ids", "selection_mode", "pending_user_selection"],
  "additionalProperties": false
}
```

### `get_selection_group_result`

description:

* `create_selectable_nodes` で作成した選択用ノード群に対する、ユーザーの選択結果を取得する

input_schema:

```json
{
  "type": "object",
  "properties": {
    "selection_group_id": { "type": "string" },
    "wait_for_user": { "type": "boolean", "default": false },
    "timeout_ms": { "type": ["integer", "null"], "minimum": 1 }
  },
  "required": ["selection_group_id"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "selection_group_id": { "type": "string" },
    "status": {
      "type": "string",
      "enum": ["pending", "selected", "expired", "cancelled"]
    },
    "selected_option_ids": { "type": "array" },
    "selected_node_ids": { "type": "array" }
  },
  "required": ["selection_group_id", "status", "selected_option_ids", "selected_node_ids"],
  "additionalProperties": false
}
```

### `submit_ask`

description:

* Ask フォーム相当の入力で新規 `RunAct` を開始する。選択中ノードは `context_node_ids` として同送できる

input_schema:

```json
{
  "type": "object",
  "properties": {
    "user_message": { "type": "string", "minLength": 1 },
    "act_type": { "type": "string", "enum": ["explore", "consult", "investigate"] },
    "topic_id": { "type": "string" },
    "workspace_id": { "type": "string" },
    "tree_id": { "type": ["string", "null"] },
    "context_node_ids": {
      "type": "array",
      "items": { "type": "string" }
    },
    "llm_config": { "type": "object" },
    "grounding_config": { "type": "object" },
    "thinking_config": { "type": "object" },
    "research_config": { "type": "object" }
  },
  "required": ["user_message", "act_type", "topic_id", "workspace_id"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "request_id": { "type": "string" },
    "accepted": { "type": "boolean" },
    "stream_state": { "type": "string", "enum": ["running", "error"] }
  },
  "required": ["request_id", "accepted", "stream_state"],
  "additionalProperties": false
}
```

### `run_act_with_context`

description:

* 起点ノードと文脈ノードを指定して派生 Act を開始する。node action の `run_act` 相当

input_schema:

```json
{
  "type": "object",
  "properties": {
    "anchor_node_id": { "type": "string" },
    "user_message": { "type": "string", "minLength": 1 },
    "context_node_ids": {
      "type": "array",
      "items": { "type": "string" }
    },
    "act_type": { "type": "string", "enum": ["explore", "consult", "investigate"] },
    "llm_config": { "type": "object" },
    "grounding_config": { "type": "object" },
    "thinking_config": { "type": "object" },
    "research_config": { "type": "object" }
  },
  "required": ["anchor_node_id", "user_message", "act_type"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "request_id": { "type": "string" },
    "accepted": { "type": "boolean" },
    "anchor_node_id": { "type": "string" }
  },
  "required": ["request_id", "accepted", "anchor_node_id"],
  "additionalProperties": false
}
```

### `set_stream_preferences`

description:

* thought 表示や grounding 利用など、stream 表示と送信の既定設定を更新する

input_schema:

```json
{
  "type": "object",
  "properties": {
    "show_thoughts": { "type": "boolean" },
    "include_thoughts": { "type": "boolean" },
    "use_web_grounding": { "type": "boolean" },
    "model_profile": {
      "type": "string",
      "enum": ["flash", "deep_research"]
    }
  },
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "show_thoughts": { "type": "boolean" },
    "include_thoughts": { "type": "boolean" },
    "use_web_grounding": { "type": "boolean" },
    "model_profile": { "type": "string" }
  },
  "required": ["show_thoughts", "include_thoughts", "use_web_grounding", "model_profile"],
  "additionalProperties": false
}
```

### `report_stream_error`

description:

* stream 中の `terminal.error`、購読例外、予期しない event を frontend の dev console と計測へ記録する

input_schema:

```json
{
  "type": "object",
  "properties": {
    "source": {
      "type": "string",
      "enum": ["terminal_error", "stream_exception", "unexpected_event", "reducer_failure"]
    },
    "request_id": { "type": ["string", "null"] },
    "trace_id": { "type": ["string", "null"] },
    "stage": { "type": ["string", "null"] },
    "retryable": { "type": ["boolean", "null"] },
    "message": { "type": "string" },
    "raw_event": { "type": ["object", "null"] }
  },
  "required": ["source", "message"],
  "additionalProperties": false
}
```

output_schema:

```json
{
  "type": "object",
  "properties": {
    "logged": { "type": "boolean" },
    "masked_fields": { "type": "array" }
  },
  "required": ["logged", "masked_fields"],
  "additionalProperties": false
}
```

## MUST / 制約

* `submit_ask` と `run_act_with_context` は `request_id` を frontend 側で生成し、同一再送時は再利用できる設計にする
* `select_nodes` と `open_node_detail` は存在しない node_id に対して `NOT_FOUND` を返す
* `create_selectable_nodes` は生成したノード群を knowledge 正本へ直接保存しない
* `get_selection_group_result` は未選択時に `pending` を返し、待機可否は caller が選べるようにする
* `report_stream_error` は token, cookie, authorization header, 個人情報を console 出力へ含めない
* read tool は state を変更しない
* write tool は UI state を変更しても知識正本を直接更新しない
* `run_act_with_context` は `anchor_node_id` を `RunActRequest.anchor` に反映する

## 実装メモ

* ADK 側には MCP 互換または同等の tool registry として公開できる形を推奨する
* frontend 実装では schema validation を入れ、未知フィールドは拒否する
* dev console 用の error 出力は `local` 優先とし、`prod` での扱いは別途固定する

## 完了条件（DoD）

* agent が frontend 能力を tool description だけで発見できる
* read/write の境界が schema で判別できる
* `RunAct` 系 UI 操作を専用ハードコードなしで tool 経由に寄せられる
