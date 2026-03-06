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
* `ACT_TYPE_CONSULT` / `ACT_TYPE_INVESTIGATE` は当面 `EXPLORE` と同等実装
* `ACT_TYPE_UNSPECIFIED` は `INVALID_ARGUMENT`

## 3.1 Vertex AI モデル方針（MUST）

* 既定モデル: Gemini 3 Flash（低レイテンシ）
* 深掘りモード: Deep Research 相当プロファイル
* 根拠が必要な問い合わせは Web Grounding を有効化
* 実際の model ID は環境変数で切替し、仕様上はプロファイル名で扱う

## 3.2 実装言語（MUST）

* バックエンド実装は Go 前提
* Connect streaming は goroutine/chan で制御

## 4. LangGraph State

| フィールド | 説明 |
| --- | --- |
| `request` | RunActRequest |
| `trace_id` | 実行トレースID |
| `context_snapshot` | 読み取り済み文脈 |
| `draft_graph` | メモリ内ドラフトグラフ |
| `planner_output` | プラン結果 |
| `model_stream_buffer` | 出力バッファ |
| `emitted_patch_count` | 送信patch数 |
| `emitted_text_chars` | 送信文字数 |
| `warnings` | 非致命警告 |
| `first_act_root_id` | 初回ACT_ROOT識別子 |
| `error` | 失敗情報 |

補足:

* `draft_graph` はメモリ上のみ保持
* follow-up時は未commit提案も prompt context に含める
* `request_id` は `(uid, workspace_id, request_id)` 冪等で扱う

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
* `NormalizePatchOps -> EmitStream`（有効PatchOpあり）
* `EmitStream -> GenerateWithModel`（stream継続）
* `GenerateWithModel -> Finalize`（モデル完了）
* 例外は `HandleError`

## 7. Nodeごとの責務

### 7.1 ValidateRequest

必須入力: `tree_id`, `act_type`, `user_message`, `workspace_id`, `uid`, `request_id`

検証ルール:

* `user_message` は trim後 1〜6000文字
* `anchor_node_ids` 最大40
* `context_node_ids` 最大120
* `request_id` はUUID形式

失敗時:

* `RunActEvent.error` を1回返して終了
* `done=true` は返さない
* 重複 `request_id` は `ALREADY_EXISTS`

### 7.2 LoadContext

* Firestoreから anchor 周辺の nodes/edges/evidence を読取
* 既定探索範囲: 2-hop
* 取得上限: nodes 500 / edges 1000 / evidence 1000

失敗時コード:

* tree不存在: `FAILED_PRECONDITION`
* 一時障害: `UNAVAILABLE`

### 7.3 BuildPrompt

* 入力: `act_type`, `user_message`, `anchor`, `context_node_ids`, snapshot
* 制約: `PatchOp(upsert/append_md)` 以外を生成しない
* 文脈空でも `ACT_ROOT` を最低1つ生成
* `llm_config.profile` と `grounding_config.use_web_grounding` を反映

### 7.4 GenerateWithModel

固定値:

* Node timeout: 60秒
* Run全体 timeout: 90秒
* モデル再試行: 最大2回（`UNAVAILABLE` / `DEADLINE_EXCEEDED`）
* 推論ループ上限: 5
* `thinking_config.include_thoughts=true` 時は thought増分を収集
* Deep Research 利用時は background実行として扱う

### 7.5 NormalizePatchOps

* 許可op: `upsert`, `append_md`
* 禁止opは破棄して warning記録
* `append_md` 対象未作成時は補完 `upsert` を先行
* 親子循環検出時は該当edge破棄
* 1 Run上限: `patch_ops <= 400`

### 7.6 EmitStream

送信規約:

* 1イベントに `patch_ops` と `text_delta` 同居可
* 送信順序は親->子
* `patch_ops` バッチ: 1〜40件
* `text_delta` バッチ: 50〜800文字
* `text_delta` 総量上限: 50,000文字
* thought増分は `stream_parts[].thought=true` で送出

ID生成規則:

* block/node id: `act_{traceId}_n{seq}`
* edge id: `act_{traceId}_e{seq}`

### 7.7 Finalize

* 正常時のみ `done=true` を1回返す
* `done=true` 後の追加送信禁止

### 7.8 HandleError

* `RunActEvent.error` を1回返す
* `done=true` は返さない
* `ErrorInfo.stage`, `retryable`, `trace_id` を必ず設定

## 8. ストリーミング規約

* 終了状態は `done` と `error` が排他
* 終了イベントは必ず最後の1イベント
* 部分成功後に失敗した場合は `error` 終了

## 9. エラーポリシー

返却コード:

* `INVALID_ARGUMENT`
* `ALREADY_EXISTS`
* `FAILED_PRECONDITION`
* `UNAVAILABLE`
* `DEADLINE_EXCEEDED`
* `INTERNAL`

運用方針:

* 入力不正は即終了
* 外部依存障害は1回再試行
* 2回連続失敗で `error` 返却

## 10. フォールバック方針

LLMが構造出力を返せない場合:

* `upsert` で `ACT_ROOT` を1件生成
* `append_md` で失敗理由を簡潔に追記
* 最後は `done=true`

## 11. Mock互換ルール

Mock `ActService` も本番と同一 `RunActEvent` 契約で返す。
