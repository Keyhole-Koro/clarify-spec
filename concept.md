# Concept (Hackathon v0.4)

このプロジェクトは、情報探索（Act）と知識整理（Organize）を分離しつつ、同じ知識グラフへ接続する。

## 目的

* 会話・検索・外部ツール情報を、抽象↔具体のノード構造で扱う
* 「探索で得た知見」を「整理された知識」へ確実に反映できる
* 根拠（evidence）を辿れる状態を維持する

## ドメイン分割

* `act/`: ユーザー対話でノード候補をリアルタイム生成（Interaction Layer）
* `organize/`: 入力を安定処理して永続知識グラフへ反映（Knowledge Pipeline Layer）
* `context/`: personalization overlay（profile/session/memory）の仕様正本
* `act/specs/behavior`: Context Assembly Layer の実行仕様
* `act/specs/contracts`: Context Bundle 契約
* `organize/specs`: topicモデルとKnowledge Pipeline仕様
* `frontend/`: 単一画面UIの表示要件と実装方針

## 不変条件（MUST）

* 構造と表示を分離する（契約 = `act/specs/contracts`、表示 = frontend）
* at-least-once前提で冪等に処理する（重複受信で壊れない）
* DraftとCommitを分離する（Actは提案、永続化はApplyPatch）
* 認証は Firebase Auth の Googleアカウント（`google.com`）に限定する
* 認可は middleware で共通化し、`token user -> workspace membership -> topic access` を毎回検証する
* workspace はユーザーが複数作成可能で、招待URLで参加できる
* 根拠参照を保持する（URL / deep link / generation / sha256）
* 知識正本キーは `topic_id` を使用する

## ストレージ責務分割（MUST）

* Firestore は確定済みの軽量メタ、関係、状態遷移、権限境界、検索キー、GCS参照ポインタを保持する
* GCS は本文、raw observation、生成物、versioned snapshot など大きい実体を保持する
* Act memory は実行中だけ有効な揮発状態とし、stream中の ephemeral node、未確定候補、途中推論を保持する
* Firestore に高頻度で揺れる stream 中状態を保存しない
* GCS の本文系オブジェクトは `gcsUri/generation/sha256` で参照可能にする
* Act memory は知識正本ではなく、確定データは Organize 経由でのみ Firestore/GCS へ昇格する
* stream 中に UI へ見えている draft は、commit 完了まで unsaved として扱う

## 3層アーキテクチャ

1. Interaction Layer（Act Run / stream）
2. Context Assembly Layer（intent -> retrieval -> ranking -> budgeting -> bundle）
3. Knowledge Pipeline Layer（Organize A0〜A7 write path）

## Act の位置づけ

* Connect RPC server-streaming で `RunAct` を実行
* Go API（Connect公開）と Python ADK Worker（推論実行）を分離する
* `PatchOp` は `upsert` / `append_md` のみ
* thought stream（thinkthrough）を通常回答と分離して表示可能
* Vertex AI（Gemini 3 Flash / Web Grounding / Deep Research）をプロファイルで使い分け
* Context Assembly は read-only で Firestore/GCS を参照

## Agent と Frontend の設計原則

* agent は frontend の機能を可能な限り `tool` として利用する
* frontend 固有の操作知識を agent 本体へ埋め込みすぎず、UI能力は tool contract として外出しする
* tool は `click` のような画面部品単位ではなく、`select_nodes` / `open_node_detail` / `run_act_with_context` のような意図単位で設計する
* 読み取り系 tool と書き込み系 tool は分離し、権限と失敗時挙動を明確にする
* frontend state は操作面のインターフェースであり、知識正本や実行契約の正本にはしない
* 新しいUI機能は、可能なら agent 専用分岐より先に再利用可能な tool として追加する

## Tool と Context Assembly の分離原則

* tool は raw input provider として、UI で観測された事実とユーザー選択を渡す
* Context Assembly は normalized model-context builder として、retrieval / ranking / budgeting / bundle 化を担当する
* tool は材料を渡し、assembly は材料を料理する
* tool は relevance score, token budget, prompt text を確定しない
* assembly は UI 表示都合ではなく、モデル入力最適化の責務を持つ
* `RunActRequest` は transport 契約に徹し、assembly 専用の内部 DTO は worker 側で生成する
* model prompt builder は tool の生データではなく assembly 後の bundle だけを見る
* MCP は tool の discovery / invocation / transport を担う公開方式であり、assembly の責務には入らない

## 論理境界の設計原則

* 物理 repo 分割より先に、`Contract` / `Act Runtime` / `Frontend Runtime` / `Organize Runtime` / `Shared Knowledge Model` の 5 境界を固定する
* `Contract` は RPC、stream、frontend tools、MCP、organize event などのサービス間契約を持ち、下位 runtime に依存しない
* `Act Runtime` はその場の実行と stream 配信を担当し、knowledge の write path を持たない
* `Frontend Runtime` は UI 表示、選択、agent-facing capability を担当し、Organize の内部処理には依存しない
* `Organize Runtime` は非同期 write path を担当し、topic/draft/bundle/graph/index の確定更新を担う
* `Shared Knowledge Model` は topic、node、edge、evidence、version、lease などの意味論の正本であり、runtime に依存しない
* 依存方向は `Frontend/Act/Organize -> Contract/Shared Knowledge Model` に限定し、runtime 同士を直接結合しない
* `frontend` が `organize` の内部仕様へ、`organize` が `frontend` の内部 state へ直接依存しない

## Organize の位置づけ

* Agent分割（A0〜A7）で入力解釈→分解→統合→索引化
* Firestoreは構造メタ、本文はGCS参照指向で管理
* lease / 冪等 / 失敗復旧を前提に運用可能にする
* topic単位で draft/outline/node/index を更新する

## UXの主軸

* フルスクリーンキャンバスでノードをストリーミング描画
* 複数選択コンテキストで派生探索
* 右ペインで詳細Markdownとアクション展開

## 仕様の正本

* Act索引: `act/specs/README.md`
* Frontend tool contract: `act/specs/contracts/frontend-agent-tools.md`
* Frontend MCP contract: `act/specs/contracts/frontend-agent-tools-mcp.md`
* Frontend tools guide: `act/readable/frontend-tools-guide.md`
* Organize索引: `organize/README.md`, `organize/specs/pipeline/summary.md`, `context/model/topic-model.md`
* Context索引: `context/README.md`
* Frontend正本: `frontend/frontend-spec.md`

## 実装時の導線

ルートの実装順と索引は `IMPLEMENTATION-INDEX.md` を参照する。
