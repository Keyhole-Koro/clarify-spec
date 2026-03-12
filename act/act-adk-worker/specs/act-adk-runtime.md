# Act ADK Runtime 仕様

Version: 1.2

本仕様は `ActService.RunAct` の内部挙動を Google ADK ベースで定義する。
外部契約は `act/specs/contracts/rpc-connect-schema.md` を正本とする。

## 目的

* `RunAct` を ADK workflow で再現可能に実行する
* thought/answer/patch のストリーミングを安定返却する
* Firestore 直接書き込みを禁止し、Draft生成に限定する

## 非目的

* Organize永続化（ApplyPatch）
* レイアウト確定
* 長期履歴DBの実装詳細

## 実行構成（MUST）

* `act-api`（Go）: Connect RPC公開、認証認可、stream変換
* `act-adk-worker`（Python + Google ADK）: Context Assembly + LLM実行

## ADK Workflow Nodes（固定）

1. `ValidateInput`
2. `ResolveIntent`
3. `RetrieveContext`
4. `RankAndBudget`
5. `GenerateWithModel`
6. `NormalizePatchOps`
7. `EmitParts`
8. `Finalize`
9. `HandleError`

## `act-adk-worker` フローチャート（詳細）

```mermaid
flowchart TD
  A[Receive Request<br>from act-api] --> B[ValidateInput]
  B -->|ok| C[ResolveIntent]
  B -->|invalid| E1[HandleError<br>INVALID_ARGUMENT]

  C --> D[RetrieveContext]
  D -->|read Firestore/GCS| D1[Context Candidates]
  D1 --> F[RankAndBudget]
  F -->|within budget| G[Build Prompt Bundle]
  F -->|over budget| F1[Truncate + Diagnostics]
  F1 --> G

  G --> H[GenerateWithModel<br>Vertex AI via ADK]
  H -->|stream chunks| I[NormalizePatchOps]
  H -->|model timeout/unavailable| E2[HandleError<br>DEADLINE_EXCEEDED/UNAVAILABLE]

  I -->|valid patch/thought/answer| J[EmitParts]
  I -->|invalid shape| E3[HandleError<br>INTERNAL]

  J --> K{stream done?}
  K -->|no| H
  K -->|yes| L[Finalize<br>emit done]

  E1 --> M[Emit error event]
  E2 --> M
  E3 --> M
```

## `act-adk-worker` 出力ストリーム変換フロー

```mermaid
flowchart LR
  MC[Model Chunk] --> P{part type}
  P -->|thought text| T[emit thought delta]
  P -->|answer text| A[emit answer delta]
  P -->|tool/grounding meta| G[attach metadata]
  T --> N[NormalizePatchOps]
  A --> N
  G --> N
  N --> E[Emit RunActEvent]
```

## `act-adk-worker` エラーフロー（分類）

```mermaid
flowchart TD
  X[Error Raised] --> Y{error source}
  Y -->|input validation| I[INVALID_ARGUMENT<br>stage=VALIDATE_REQUEST<br>retryable=false]
  Y -->|context retrieval| C[UNAVAILABLE<br>stage=ASSEMBLY_RETRIEVE<br>retryable=true]
  Y -->|budgeting| B[RESOURCE_EXHAUSTED or degrade<br>stage=ASSEMBLY_BUDGET]
  Y -->|vertex call| V[UNAVAILABLE/DEADLINE_EXCEEDED<br>stage=GENERATE_WITH_MODEL<br>retryable=true]
  Y -->|patch normalize| N[INTERNAL<br>stage=NORMALIZE_PATCH_OPS<br>retryable=false]
```

## State（論理）

| フィールド | 説明 |
| --- | --- |
| `request` | RunActRequest |
| `trace_id` | 実行トレースID |
| `intent` | 解釈済みintent |
| `prompt_bundle` | Assembly結果 |
| `model_stream_buffer` | 増分バッファ |
| `emitted_patch_count` | 送信patch数 |
| `warnings` | 非致命警告 |
| `error` | 失敗情報 |

## 正常フロー

1. `ValidateInput`: 入力検証
2. `ResolveIntent`: intent決定
3. `RetrieveContext`: Firestore/GCS read-only取得
4. `RankAndBudget`: 文脈圧縮
5. `GenerateWithModel`: ADKでモデル実行（chunk単位）
6. `NormalizePatchOps`: `upsert|append_md` 制約適用
7. `EmitParts`: thought/answer/patch を順次送信
8. `Finalize`: `done=true` 終端

## ノード別 I/O（実装指針）

| Node | 入力 | 出力 | MUST |
| --- | --- | --- | --- |
| `ValidateInput` | RunActRequest | validated request | `topic_id` 必須、空query拒否 |
| `ResolveIntent` | query, selected nodes | intent, retrieval policy | intent未判定時は `explore` へフォールバック |
| `RetrieveContext` | topic_id, policy | candidates | Firestore/GCS read-only |
| `RankAndBudget` | candidates, token budget | compacted context, diagnostics | drop理由を diagnostics に残す |
| `GenerateWithModel` | prompt bundle | model chunks | timeout/retry を守る |
| `NormalizePatchOps` | chunks | PatchOp events | `op in {upsert, append_md}` のみ |
| `EmitParts` | normalized events | stream frames | `trace_id` を全frameに付与 |
| `Finalize` | stream state | done frame | errorとdoneを同時送信しない |
| `HandleError` | internal error | error frame | `code/stage/retryable` を埋める |

## 異常フロー（error/retryable/stage）

* 入力不正: `INVALID_ARGUMENT`, `stage=VALIDATE_REQUEST`
* assembly失敗: `ASSEMBLY_*`（retryableは条件依存）
* モデル障害: `UNAVAILABLE|DEADLINE_EXCEEDED`, `stage=GENERATE_WITH_MODEL`
* 形式不正: `INTERNAL`, `stage=NORMALIZE_PATCH_OPS`

## cancel 伝播（MUST）

* worker は `act-api` から受けた request context を最上位の cancellation source として扱う
* `GenerateWithModel`, grounding, 外部 tool 呼び出し, polling, normalize は同一 context またはその子 context を受け取る
* cancel 検知後は新しい chunk / patch / thought / answer を生成しない
* 既に emit 済みの event は巻き戻さない
* client disconnect 起因では error frame を追加生成しない
* cleanup は best-effort とし、cancel 伝播の完了を block しない
* cooperative cancel 非対応の SDK/外部 tool には短い timeout を併用して停止を補助する

## stream 順序保証（MUST）

* 初回の本文系 `append_md` より先に、対象 block の `upsert` を生成する
* 同一 `RunActEvent` 内の基本順序は `thought -> answer -> upsert -> append_md -> metadata -> terminal` とする
* `append_md` は対象 block の `upsert` より先行してはならない
* `append_md` は空文字を生成しない
* metadata は本文生成を block せず、answer より先に必須化しない
* `thought` は Markdown 本文 patch と混ぜない
* `done/error` は最後の frame にのみ現れ、終端後の追加 frame を生成しない

## model profile 切替（MUST）

* 既定 profile は `Flash` とする
* `request.research_config.use_deep_research=true` の場合のみ `Deep Research` を選択する
* `request.grounding_config.enabled=true` の場合は grounding を独立に有効化する
* `Deep Research` 実行が timeout / `UNAVAILABLE` / 連続 5xx で継続不能になった場合は、同一 request 内で `Flash` へ fallback する
* profile の最終決定は worker が行い、frontend の UI 表示値をそのまま内部 profile 名に直結させない

## metadata 正規化（MUST）

* `grounding metadata` は frontend 表示に必要な最小 shape に正規化する
  * `references[] { title, url, host, snippet? }`
* `tool metadata` は要約 shape に正規化する
  * `tools[] { tool_name, status, summary, started_at?, finished_at? }`
* `diagnostics` には `model_profile`, `grounding_used`, `fallback_used`, `fallback_reason`, `trace_id` を載せられる
* SDK 依存の raw payload 全量は `RunActEvent.metadata` に載せない
* 大きな JSON payload, 内部 token/debug 情報, chain-of-thought 相当の内部詳細は外部契約へ出さない
* raw 詳細は server log / internal trace に保持する

## 数値パラメータ

* run timeout: 90s
* model retry: 2
* patch_ops max: 400
* thought flush: 500ms

## 受け入れ条件（DoD）

* ADK workflow で `RunAct` を end-to-end 説明できる
* `RunActEvent` 契約（done/error排他）を守る
* read-only境界（Assembly/ADK）を守る
* cancel が worker 下流処理まで伝播し、切断後に event を追加生成しない
* stream 順序が安定し、frontend reducer が順不同前提を強いられない

## 実装メモ（最小）

* ADK未対応機能は限定的RESTラッパーを許容
* Go側は常に契約変換レイヤとして残す
