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
  LlmConfig llm_config = 9;
  GroundingConfig grounding_config = 10;
  ThinkingConfig thinking_config = 11;
  ResearchConfig research_config = 12;
}

message ErrorInfo {
  string code = 1;
  string message = 2;
}

message RunActEvent {
  // Optional plain text stream for progressive UX.
  string text_delta = 1;
  repeated PatchOp patch_ops = 2;
  repeated StreamTextPart stream_parts = 3;

  oneof terminal {
    bool done = 10;
    ErrorInfo error = 11;
  }
}
```

## 2.1 LLM / Grounding Config

```proto
enum LlmProfile {
  LLM_PROFILE_UNSPECIFIED = 0;
  LLM_PROFILE_GEMINI_3_FLASH = 1;
  LLM_PROFILE_GEMINI_DEEP_RESEARCH = 2;
}

message LlmConfig {
  LlmProfile profile = 1;
  string model = 2;
  float temperature = 3;
  int32 max_output_tokens = 4;
}

message GroundingConfig {
  bool use_web_grounding = 1;
  int32 max_sources = 2;
}

message ThinkingConfig {
  // Maps to Gemini includeThoughts.
  bool include_thoughts = 1;
}

message ResearchConfig {
  // Deep Research execution intent.
  bool use_deep_research = 1;
  bool background = 2;
}

message StreamTextPart {
  string text = 1;
  bool thought = 2;
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
  // For structured data not suitable for props (e.g. grounding sources)
  google.protobuf.Struct metadata = 8;
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
* 既定LLMは `GEMINI_3_FLASH`、重探索時は `GEMINI_DEEP_RESEARCH` を許可
* Web Groundingは `grounding_config.use_web_grounding=true` で有効化
* thoughtストリームは `stream_parts[].thought=true` で返す
* 既存互換のため通常テキストは `text_delta` でも返してよい

## 5. Frontend Acceptance Mapping

* `upsert` -> storeへ block追加/更新
* `append_md` -> `contentMd` 追記
* `stream_parts[].thought=true` -> Thinkthroughパネルへ追記
* `stream_parts[].thought=false` -> 通常回答として追記
* `error` -> UIに失敗表示、loading解除
* `done=true` -> loading解除

## 6. References

* `act/specs/behavior/act-flow.md`
* `act/specs/behavior/act-langgraph-runtime.md`
* `act/specs/quality/frontend-stream-acceptance.md`
* `act/specs/contracts/gemini-vertex-response-schemas.md`
