# Act LangGraph Runtime 仕様

Version: 1.1

本仕様は `ActService.RunAct` の内部挙動を LangGraph ベースで定義する。
外部契約は `act/specs/contracts/rpc-connect-schema.md` を正本とする。

## 1. 目的

* `RunAct` を再現可能な状態遷移として固定する
* ストリーミング（`text_delta`, `patch_ops`）を安定して返す
* Firestoreへの直接書き込みを禁止し、Draft生成に限定する

## 2. 非目的

* 永続化処理（`OrganizeService.ApplyPatch` が担当）
* レイアウト確定（`layout_locked` の変更）
* 長期履歴管理（Run単位の短期状態のみ）

## 3. MVPスコープ固定（ハッカソン運用）

* 本実装対象: `ACT_TYPE_EXPLORE`
* `ACT_TYPE_CONSULT` / `ACT_TYPE_INVESTIGATE` は当面 `EXPLORE` と同じ実装にフォールバック
* `ACT_TYPE_UNSPECIFIED` は `INVALID_ARGUMENT`

## 3.1 Vertex AI モデル方針（MUST）

* 既定モデル: Gemini 3 Flash（低レイテンシ）
* 深掘りモード: Deep Research 相当プロファイルを使用
* Web根拠が必要な問い合わせは Web Grounding を有効化
* 実際の model ID は環境変数で切替し、仕様上はプロファイル名で扱う

## 3.2 実装言語（MUST）

* バックエンド実装は Go を前提とする
* Connect streaming は Go の goroutine/chan で実装する

## 4. LangGraph State

```txt
ActState {
  request: RunActRequest
  trace_id: string
  context_snapshot: TreeContext
  draft_graph: DraftGraph
  planner_output: Plan
  model_stream_buffer: string
  emitted_patch_count: number
  emitted_text_chars: number
  warnings: string[]
  first_act_root_id?: string
  error?: { code, message }
}
```

補足:

* `draft_graph` はメモリ上のみ保持
* follow-up時は未commit提案も prompt context に含める

## 5. Graph Nodes（固定）

1. `ValidateRequest`
2. `LoadContext`
3. `BuildPrompt`
4. `GenerateWithModel`
5. `NormalizePatchOps`
6. `EmitStream`
7. `Finalize`
8. `HandleError`

## 6. Edge 条件

* `ValidateRequest -> LoadContext`（必須項目が有効）
* `LoadContext -> BuildPrompt`（snapshot取得成功）
* `BuildPrompt -> GenerateWithModel`
* `GenerateWithModel -> NormalizePatchOps`（chunkまたは構造出力受信）
* `NormalizePatchOps -> EmitStream`（有効 `PatchOp` あり）
* `EmitStream -> GenerateWithModel`（stream継続）
* `GenerateWithModel -> Finalize`（モデル完了）
* 例外はすべて `HandleError`

## 7. Nodeごとの責務

### 7.1 ValidateRequest

必須入力:

* `tree_id`
* `act_type`
* `user_message`
* `workspace_id`
* `uid`

検証ルール:

* `user_message` は trim 後 1〜4000 文字
* `anchor_node_ids` は最大 20
* `context_node_ids` は最大 50

失敗時:

* `RunActEvent.error` を1回返して終了
* `done=true` は返さない

### 7.2 LoadContext

* Firestoreから anchor 周辺の nodes/edges/evidence を読み込み
* 既定探索範囲: 2-hop
* 取得上限: nodes 200 / edges 400 / evidence 400

失敗時コード:

* tree不存在: `FAILED_PRECONDITION`
* 一時障害: `UNAVAILABLE`

### 7.3 BuildPrompt

* 入力: `act_type`, `user_message`, `anchor`, `context_node_ids`, snapshot
* モデル命令は「`PatchOp(upsert/append_md)` 以外を生成しない」制約を強制
* 文脈が空の場合でも `ACT_ROOT` を最低1つ生成する方針を指示
* `llm_config.profile` と `grounding_config.use_web_grounding` をプロンプト方針へ反映

### 7.4 GenerateWithModel

固定値:

* Node timeout: 30秒
* Run全体 timeout: 45秒
* モデル再試行: 最大1回（`UNAVAILABLE` / `DEADLINE_EXCEEDED` のみ）
* 推論ループ上限: 3サイクル
* `GEMINI_DEEP_RESEARCH` 利用時は timeout超過を優先監視
* `thinking_config.include_thoughts=true` 時は thought増分を収集
* Deep Research 利用時は Interactions API を background 実行で扱う

### 7.5 NormalizePatchOps

* 許可op: `upsert`, `append_md` のみ
* 禁止opは破棄し `warnings` に記録
* `append_md` 対象 block 未作成時は補完 `upsert` を先行追加
* 親子循環検出時は該当edgeを破棄
* 1 Run あたり上限: `patch_ops <= 200`

### 7.6 EmitStream

送信規約:

* 1イベントに `patch_ops` と `text_delta` の同居を許可
* 送信順序は親 -> 子
* `patch_ops` バッチ: 1〜20件
* `text_delta` バッチ: 50〜400文字
* `text_delta` 総量上限: 20,000 文字
* thought増分は `stream_parts[].thought=true` で送出

ID生成規則（固定）:

* block/node id: `act_{traceId}_n{seq}`
* edge id: `act_{traceId}_e{seq}`

### 7.7 Finalize

* 正常時のみ `done=true` を1回返す
* `done=true` 送信後の追加イベント送信を禁止
* メトリクス記録後に終了

### 7.8 HandleError

* `RunActEvent.error` を1回返す
* `done=true` は返さない
* 可能なら `warnings` をログへ残す

## 8. ストリーミング規約（RunActEvent）

* 終了状態は `done` と `error` が排他的
* 終了イベントは必ず最後の1イベント
* 部分成功後に失敗した場合は `error` 終了

## 9. エラーポリシー

返却コード（`ErrorInfo.code`）:

* `INVALID_ARGUMENT`
* `FAILED_PRECONDITION`
* `UNAVAILABLE`
* `DEADLINE_EXCEEDED`
* `INTERNAL`

運用方針:

* 入力不正は即終了
* 外部依存障害は1回再試行
* 2回連続失敗で `error` 返却

## 10. フォールバック方針（固定）

LLMが構造出力を返せない場合でも、以下を返してUIを壊さない。

* `upsert` で `ACT_ROOT` を1件生成
* `append_md` で失敗理由を簡潔に追記
* 最後は `done=true`

## 11. Mock互換ルール（固定）

Mock `ActService` は本番と同一契約で返す。

* `RunActEvent` フォーマットを一致
* `upsert` / `append_md` 以外を返さない
* `done` / `error` 排他を守る
* `ACT_ROOT` を必ず含める

## 12. 観測性

最低限のメトリクス:

* `run_act_total`
* `run_act_error_total`
* `run_act_latency_ms`
* `emitted_patch_ops_total`
* `emitted_text_chars_total`

ログ必須項目:

* `traceId`, `treeId`, `uid`, `actType`, `anchorNodeIds`, `error.code`, `latencyMs`

## 13. 仕様整合チェック

* Firestore直接更新はしない
* `PatchOp` は `upsert` / `append_md` のみ
* レイアウト固定はユーザー操作経由のみ
* 永続化は `OrganizeService.ApplyPatch` のみ
