# キャンバス実装フェーズ（Phase 1〜4）

本資料は、ReactFlowベースの単一画面SPAで「思考の生成・接続・深掘り」を段階実装するための要件定義である。
前提仕様は `frontend/frontend-spec.md` と `act/specs/behavior/act-flow.md` を正本とする。

## 共通前提

* Next.js App Router（SPA運用）
* ReactFlow + Zustand + Tailwind CSS
* Markdown描画は `react-markdown` + `rehype-sanitize`
* RPCストリームで `PatchOp` を受信し、UIを逐次更新
* ReactFlow実装は Client Component 前提（必要なら `dynamic(..., { ssr: false })`）

## Phase 1: キャンバス土台とストリーミング描画

目的:

* Askフォーム送信から、RPCストリーム受信、キャンバスへのリアルタイム描画までを通す

必須実装:

* `features/knowledgeTree/store.ts`
* `features/actExplore/hooks/useActStream.ts`
* `features/graph/components/GraphCanvas.tsx`
* `features/graph/components/CustomNode.tsx`
* `components/ui/AskForm.tsx`

受け入れ条件:

* `upsert` でブロック追加/更新できる
* `append_md` で `contentMd` 追記が連続反映される
* Block親子を ReactFlow `nodes`/`edges` に最低限マップできる
* 画面下部中央の Askフォームから送信できる

## Phase 2: 複数選択コンテキストと文脈エッジ

目的:

* 複数ノード選択をAI入力コンテキストに載せ、生成ノードへ依存エッジを引く

必須実装:

* GraphCanvasの選択機能（box/lasso）を有効化
* Zustandに `selectedNodeIds` と `contextEdges` を追加
* Ask送信時に選択ノード本文を文脈として同送
* 生成開始時（新規ACT_ROOT受信時）に `selectedNodeIds -> newActRoot` エッジを追加
* 送信後に `selectedNodeIds` をクリア

受け入れ条件:

* ドラッグ複数選択が可能
* 選択中ノードの視覚ハイライトが有効
* 新規生成ノードに対して文脈エッジが描画される

## Phase 3: 右ペイン詳細表示とrun_act展開

目的:

* ノード詳細表示と、ノード内アクションからの派生生成を実現する

必須実装:

* Zustandに `activeNodeId` を追加
* `features/nodeMarkdown/components/MarkdownPane.tsx` を実装
* GraphCanvasから `onNodeClick` または `onNodeDoubleClick` で `activeNodeId` 設定
* CustomNodeで `actions[]` を描画し、`on_click.type === "run_act"` を実行
* run_act実行時に `anchor` へ起点ノードIDを必ず設定
* 起点ノードから生成ノードへの依存エッジを追加

受け入れ条件:

* ノード選択時に右ペインへ詳細Markdownが表示される
* run_actボタンで新たなストリーム生成が始まる
* 派生関係がキャンバス上の線で追える

## Phase 4: Agent Tool Interaction UI

目的:

* frontend agent tools に対応する selection group UI と agent interaction を実装する

必須実装:

* selection group UI の詳細は `act/specs/behavior/frontend-agent-tool-interactions.md` に従う
* `create_selectable_nodes` / `get_selection_group_result` / `report_stream_error` に対応する state と UI を追加する

受け入れ条件:

* `act/specs/quality/frontend-agent-tools-acceptance.md` を満たす

## 実装順序（推奨）

1. Phase 1を完了し、ストリーム描画を安定化
2. Phase 2で選択コンテキストと依存エッジを追加
3. Phase 3で詳細表示とアクション分岐を追加
4. Phase 4で agent 選択ノードとユーザー選択UIを追加

## 仕様整合メモ

* `append_md` は常に追記（置換しない）
* `parent_id` は構造ツリー、`contextEdges` は文脈依存として分離
* 永続化は `ApplyPatch` 経由のみ（Draft直接書き込み禁止）

## 実装チェック参照

* `act/specs/quality/frontend-stream-acceptance.md`
* `act/specs/quality/frontend-agent-tools-acceptance.md`
