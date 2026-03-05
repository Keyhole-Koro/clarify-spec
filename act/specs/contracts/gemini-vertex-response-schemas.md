# Gemini / Vertex AI レスポンススキーマ集約

Version: v1  
Scope: Act backend（Go）で扱う外部LLMレスポンス

## 1. Embedding

### 1.1 Gemini Developer API `models.embedContent`

```json
{
  "embedding": {
    "values": [0.0123, -0.0456, 0.0789]
  }
}
```

Batch:

```json
{
  "embeddings": [
    { "values": [0.1, 0.2] },
    { "values": [0.3, 0.4] }
  ]
}
```

### 1.2 Vertex AI `models.embedContent`

```json
{
  "embedding": {
    "values": [0.0123, -0.0456, 0.0789]
  },
  "usageMetadata": {
    "promptTokenCount": 123,
    "candidatesTokenCount": 0,
    "totalTokenCount": 123
  },
  "truncated": false
}
```

Act利用方針:

* EmbeddingはA1/A4用途が中心。Act内では将来のretrieval拡張で利用。

## 2. Web Grounding

### 2.1 Developer API（GenerateContent）

```json
{
  "candidates": [
    {
      "content": { "parts": [{ "text": "..." }], "role": "model" },
      "groundingMetadata": {
        "webSearchQueries": ["..."],
        "searchEntryPoint": { "renderedContent": "<div>...</div>" },
        "groundingChunks": [
          { "web": { "uri": "https://...", "title": "example.com" } }
        ],
        "groundingSupports": [
          {
            "segment": { "startIndex": 0, "endIndex": 85, "text": "..." },
            "groundingChunkIndices": [0]
          }
        ]
      }
    }
  ]
}
```

### 2.2 Vertex AI `GroundingMetadata`

代表フィールド:

* `webSearchQueries[]`
* `imageSearchQueries[]`
* `retrievalQueries[]`
* `groundingChunks[]`
* `groundingSupports[]`
* `searchEntryPoint.renderedContent`
* `retrievalMetadata`
* `googleMapsWidgetContextToken`

Act利用方針:

* UIには簡約して返す（source URL/title, segment）
* 詳細メタデータはサーバログまたは内部構造に保持

## 3. Thinking Stream（Gemini）

`includeThoughts=true` のとき、partsに thoughtフラグ付き増分が来る。

```ts
for await (const chunk of response) {
  for (const part of chunk.candidates[0].content.parts ?? []) {
    if (!part.text) continue;
    if (part.thought) {
      // thought
    } else {
      // answer
    }
  }
}
```

Actマッピング:

* `part.thought=true` -> `RunActEvent.stream_parts[].thought=true`
* `part.thought=false` -> `RunActEvent.stream_parts[].thought=false`
* 既存互換として回答は `text_delta` にも反映可

## 4. Deep Research（Interactions API）

### 4.1 Interaction Resource

```json
{
  "id": "v1_...",
  "agent": "deep-research-pro-preview-12-2025",
  "status": "completed",
  "object": "interaction",
  "created": "2025-11-26T12:22:47Z",
  "updated": "2025-11-26T12:22:47Z",
  "role": "agent",
  "outputs": [{ "type": "text", "text": "..." }],
  "usage": {
    "total_input_tokens": 20,
    "total_output_tokens": 1000,
    "total_thought_tokens": 500,
    "total_tokens": 1520
  }
}
```

主要 `status`:

* `in_progress`
* `requires_action`
* `completed`
* `failed`
* `cancelled`
* `incomplete`

### 4.2 SSEイベント

* `interaction.start`
* `interaction.status_update`
* `content.start`
* `content.delta`
* `content.stop`
* `interaction.complete`

Act利用方針:

* Deep Researchは background前提
* `interaction.complete` までポーリング/SSEで待機
* 途中 `content.delta` は `stream_parts` へ橋渡し可能

## 5. Go実装での扱い

* Vertex AI SDKレスポンスを内部DTOへ正規化
* 正規化DTOから `RunActEvent` へ変換
* `usageMetadata` / `usage.total_thought_tokens` をメトリクスへ反映

## 6. Contract Alignment

本ドキュメントは以下と整合する。

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/runact-implementation.md`
* `act/specs/quality/e2e-test-strategy.md`
