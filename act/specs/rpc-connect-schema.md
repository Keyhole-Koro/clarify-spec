# Act Connect RPC Schema (Markdown)

Version: v1  
Owner: Act Domain

この文書は Act の Connect RPC 契約を Markdown で定義する。

## 1. Service Contract

```proto
syntax = "proto3";

package clarify.act.v1;

service ActService {
  // Read-only: draft patch stream only.
  rpc RunAct(RunActRequest) returns (stream RunActEvent);
}
```

## 2. Messages

```proto
message RunActRequest {
  string tree_id = 1;
  ActType act_type = 2;
  Anchor anchor = 3;
  repeated string anchor_node_ids = 4;
  string user_message = 5;
  repeated string context_node_ids = 6;
  string workspace_id = 7;
  string uid = 8;
}

message ErrorInfo {
  string code = 1;
  string message = 2;
}

message RunActEvent {
  // Optional plain text stream for progressive UX.
  string text_delta = 1;
  repeated PatchOp patch_ops = 2;

  oneof terminal {
    bool done = 10;
    ErrorInfo error = 11;
  }
}
```

## 3. Shared Types (Act Scope)

```proto
enum ActType {
  ACT_TYPE_UNSPECIFIED = 0;
  ACT_TYPE_EXPLORE = 1;
  ACT_TYPE_CONSULT = 2;
  ACT_TYPE_INVESTIGATE = 3;
}

message Anchor {
  string act_node_id = 1;
  string parent_node_id = 2;
}

enum BlockKind {
  BLOCK_KIND_UNSPECIFIED = 0;
  BLOCK_KIND_ACT_ROOT = 1;
  BLOCK_KIND_MARKDOWN = 2;
  BLOCK_KIND_SECTION = 3;
  BLOCK_KIND_LIST = 4;
  BLOCK_KIND_NODE = 5;
}

message Block {
  string id = 1;
  BlockKind kind = 2;
  string parent_id = 3;
  string title = 4;
  string content_md = 5;
  repeated Action actions = 6;
  map<string, string> props = 7;
}

message Action {
  string id = 1;
  string label = 2;
  ActionOnClick on_click = 3;
}

message ActionOnClick {
  oneof kind {
    RunActClickPayload run_act = 1;
  }
}

message RunActClickPayload {
  ActType act = 1;
  Anchor anchor = 2;
  string user_message = 3;
}

message PatchOp {
  oneof op {
    UpsertPatch upsert = 1;
    AppendMdPatch append_md = 2;
  }
}

message UpsertPatch {
  repeated Block blocks = 1;
}

message AppendMdPatch {
  string block_id = 1;
  string delta = 2;
}
```

## 4. Contract Rules (MUST)

* `RunAct` は read-only（Firestore直接更新禁止）
* `patch_ops` は `upsert` / `append_md` のみ
* `done` と `error` は排他
* `done=true` または `error` を最後に1回返して終了

## 5. Frontend Acceptance Mapping

* `upsert` -> storeへ block追加/更新
* `append_md` -> `contentMd` 追記
* `error` -> UIに失敗表示、loading解除
* `done=true` -> loading解除

## 6. References

* `act/specs/act-flow.md`
* `act/backend/act-langgraph-spec.md`
* `act/frontend/act-stream-acceptance.md`
