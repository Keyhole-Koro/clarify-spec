# Gemini / Vertex AI レスポンススキーマ集約

Version: v1  
Scope: Act backend（Go）で扱う外部LLMレスポンス

## 目的

Gemini / Vertex 系APIの返却要素を、実装コードなしで参照できる形に固定する。

## スコープ / 非スコープ

* スコープ: Embedding, Grounding, Thinking, Deep Research の返却フィールド
* 非スコープ: SDK実装サンプル、言語別コード

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runact-implementation.md`

## 1. Embedding

### 1.1 Developer API

| フィールド | 説明 |
| --- | --- |
| `embedding.values[]` | 埋め込みベクトル |
| `embeddings[]` | Batch時の埋め込み配列 |

### 1.2 Vertex AI

| フィールド | 説明 |
| --- | --- |
| `embedding.values[]` | 埋め込みベクトル |
| `usageMetadata.promptTokenCount` | 入力トークン |
| `usageMetadata.totalTokenCount` | 総トークン |
| `truncated` | 入力切り詰め有無 |

Act利用方針:

* Embeddingは将来のretrieval拡張用途
* 現時点では主にOrganize側の索引強化に利用

## 2. Web Grounding

| フィールド | 説明 |
| --- | --- |
| `groundingMetadata.webSearchQueries[]` | 実行検索クエリ |
| `groundingMetadata.groundingChunks[]` | 参照ソース（URI/Title） |
| `groundingMetadata.groundingSupports[]` | 回答断片と根拠の対応 |
| `groundingMetadata.searchEntryPoint` | 検索UI補助情報 |

Act利用方針:

* UIには source URL/title と該当segmentを簡約表示
* 詳細メタデータはログまたは内部DTOに保持

## 3. Thinking Stream（includeThoughts）

| 判定 | 意味 |
| --- | --- |
| thought=true | 思考過程テキスト |
| thought=false | 通常回答テキスト |

Actマッピング:

* thought=true -> `RunActEvent.stream_parts[].thought=true`
* thought=false -> `RunActEvent.stream_parts[].thought=false`
* 既存互換のため通常回答は `text_delta` にも反映可

## 4. Deep Research（Interactions API）

### 4.1 Interaction Resource 主要項目

| フィールド | 説明 |
| --- | --- |
| `id` | Interaction識別子 |
| `agent` | 利用エージェント |
| `status` | 実行状態 |
| `outputs` | 出力コンテンツ |
| `usage` | token使用量 |

### 4.2 status

* `in_progress`
* `requires_action`
* `completed`
* `failed`
* `cancelled`
* `incomplete`

### 4.3 SSEイベント種別

* `interaction.start`
* `interaction.status_update`
* `content.start`
* `content.delta`
* `content.stop`
* `interaction.complete`

Act利用方針:

* Deep Researchは background前提
* `interaction.complete` まで待機し、必要に応じて `content.delta` を stream_parts に橋渡し

## 5. 運用ルール（MUST）

* 外部レスポンスは内部DTOへ正規化してから `RunActEvent` へ変換
* usage系フィールドは observability 指標へ反映
* レスポンス構造変更時はこの文書を先に更新

## 6. 参照

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/quality/llm-api-test-spec.md`
