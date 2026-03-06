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
5. `GenerateWithModel`: ADKでモデル実行
6. `NormalizePatchOps`: op制約適用
7. `EmitParts`: thought/answer/patch送信
8. `Finalize`: done終端

## 異常フロー（error/retryable/stage）

* 入力不正: `INVALID_ARGUMENT`, `stage=VALIDATE_REQUEST`
* assembly失敗: `ASSEMBLY_*`（retryableは条件依存）
* モデル障害: `UNAVAILABLE|DEADLINE_EXCEEDED`, `stage=GENERATE_WITH_MODEL`
* 形式不正: `INTERNAL`, `stage=NORMALIZE_PATCH_OPS`

## 数値パラメータ

* run timeout: 90s
* model retry: 2
* patch_ops max: 400
* thought flush: 500ms

## 受け入れ条件（DoD）

* ADK workflow で `RunAct` を end-to-end 説明できる
* `RunActEvent` 契約（done/error排他）を守る
* read-only境界（Assembly/ADK）を守る

## 実装メモ（最小）

* ADK未対応機能は限定的RESTラッパーを許容
* Go側は常に契約変換レイヤとして残す
