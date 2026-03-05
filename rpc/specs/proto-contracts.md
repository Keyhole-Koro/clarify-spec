# RPC Proto Contracts (Markdown Source of Truth)

実体 `.proto` ファイルは持たず、このMarkdownを契約の正本とする。

## common/v1/types.proto

```proto
syntax = "proto3";

package clarify.common.v1;

option go_package = "github.com/clarify-spec/rpc/gen/go/common/v1;commonv1";

message Anchor {
  string act_node_id = 1;
  string parent_node_id = 2;
}

enum ActType {
  ACT_TYPE_UNSPECIFIED = 0;
  ACT_TYPE_EXPLORE = 1;
  ACT_TYPE_CONSULT = 2;
  ACT_TYPE_INVESTIGATE = 3;
}

enum BlockKind {
  BLOCK_KIND_UNSPECIFIED = 0;
  BLOCK_KIND_ACT_ROOT = 1;
  BLOCK_KIND_MARKDOWN = 2;
  BLOCK_KIND_SECTION = 3;
  BLOCK_KIND_LIST = 4;
  BLOCK_KIND_NODE = 5;
}

message RunActClickPayload {
  ActType act = 1;
  Anchor anchor = 2;
  string user_message = 3;
}

message ActionOnClick {
  oneof kind {
    RunActClickPayload run_act = 1;
  }
}

message Action {
  string id = 1;
  string label = 2;
  ActionOnClick on_click = 3;
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

message UpsertPatch {
  repeated Block blocks = 1;
}

message AppendMdPatch {
  string block_id = 1;
  string delta = 2;
}

message PatchOp {
  oneof op {
    UpsertPatch upsert = 1;
    AppendMdPatch append_md = 2;
  }
}

message Layout {
  double x = 1;
  double y = 2;
}

message Evidence {
  string evidence_id = 1;
  string type = 2;
  string title = 3;
  string url = 4;
  string ref = 5;
  string excerpt = 6;
}

message Node {
  string node_id = 1;
  string title = 2;
  string content_md = 3;
  bool layout_locked = 4;
  Layout layout = 5;
}

message Edge {
  string edge_id = 1;
  string parent_id = 2;
  string child_id = 3;
  int64 order_key = 4;
}
```

## act/v1/act.proto

```proto
syntax = "proto3";

package clarify.act.v1;

option go_package = "github.com/clarify-spec/rpc/gen/go/act/v1;actv1";

import "common/v1/types.proto";

service ActService {
  rpc RunAct(RunActRequest) returns (stream RunActEvent);
}

message RunActRequest {
  string tree_id = 1;
  clarify.common.v1.ActType act_type = 2;
  clarify.common.v1.Anchor anchor = 3;
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
  string text_delta = 1;
  repeated clarify.common.v1.PatchOp patch_ops = 2;

  oneof terminal {
    bool done = 10;
    ErrorInfo error = 11;
  }
}
```

## organize/v1/organize.proto

```proto
syntax = "proto3";

package clarify.organize.v1;

option go_package = "github.com/clarify-spec/rpc/gen/go/organize/v1;organizev1";

import "common/v1/types.proto";

service OrganizeService {
  rpc ApplyPatch(ApplyPatchRequest) returns (ApplyPatchResponse);
  rpc GetSnapshot(GetSnapshotRequest) returns (GetSnapshotResponse);
  rpc UpdateNodeFields(UpdateNodeFieldsRequest) returns (UpdateNodeFieldsResponse);
}

message ApplyPatchRequest {
  string tree_id = 1;
  string mutation_id = 2;
  repeated clarify.common.v1.PatchOp patch_ops = 3;
  string workspace_id = 4;
  string uid = 5;
}

message ApplyPatchResponse {
  repeated string changed_node_ids = 1;
  repeated string changed_edge_ids = 2;
}

message GetSnapshotRequest {
  string tree_id = 1;
  string workspace_id = 2;
  string uid = 3;
}

message GetSnapshotResponse {
  string tree_id = 1;
  string title = 2;
  repeated string entry_node_ids = 3;
  repeated clarify.common.v1.Node nodes = 4;
  repeated clarify.common.v1.Edge edges = 5;
}

message UpdateNodeFieldsRequest {
  string tree_id = 1;
  string node_id = 2;
  optional string title = 3;
  optional string content_md = 4;
  optional bool layout_locked = 5;
  optional clarify.common.v1.Layout layout = 6;
  string workspace_id = 7;
  string uid = 8;
}

message UpdateNodeFieldsResponse {
  clarify.common.v1.Node node = 1;
}
```
