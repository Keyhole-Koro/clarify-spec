# Frontend Tools Guide（Human View）

## これは何か

Act では、agent が frontend の UI 機能を直接知っている前提にしない。  
代わりに frontend の能力を `tool` として公開し、agent はその contract を見て使う。

要するに、

* agent は「画面をベタ書きで操作する知能」ではない
* frontend は「agent が使える道具箱」を提供する

という分離にしている。

## なぜこの形にするか

### 1. 汎用化しやすい

agent 本体に ReactFlow や右ペインの知識を埋め込まなくてよい。  
agent を差し替えても、同じ tool contract を読めれば同じ UI 能力を使える。

### 2. 責務がきれい

frontend は UI 状態と操作を持つ。  
agent は「何をしたいか」を決め、tool を呼ぶ。  
知識正本や stream 契約は別仕様に残す。

### 3. 観測しやすい

「ユーザーが選択した」のか「agent が選択した」のかを、tool 呼び出しとして追跡しやすい。

## どういう粒度で tool を切るか

低レベルな `click_button` ではなく、意図単位で切る。

良い例:

* `get_visible_graph`
* `get_selected_nodes`
* `open_node_detail`
* `create_selectable_nodes`
* `submit_ask`
* `run_act_with_context`

避けたい例:

* `click_canvas`
* `click_right_pane_tab`
* `toggle_button_by_selector`

理由は、低レベル tool だと UI 実装変更に弱く、agent 側も brittle になるから。

## read tool と write tool

### read tool

frontend の現在状態を読む。

例:

* `get_visible_graph`
* `get_selected_nodes`
* `get_active_node_detail`

これは state を変えない。

### write tool

frontend の操作状態や実行を更新する。

例:

* `select_nodes`
* `open_node_detail`
* `create_selectable_nodes`
* `submit_ask`
* `run_act_with_context`
* `set_stream_preferences`

これは UI state を変えるが、知識正本を直接更新しない。

## 派生 Act とは何か

派生 Act は「別 agent を常駐起動する」という意味ではなく、  
あるノードを起点にした新しい `RunAct` を分岐実行すること。

イメージ:

* 元ノードを `anchor` にする
* 必要なら選択中ノードを `context_node_ids` に載せる
* 新しい `request_id` で別 stream を開始する

UI 上は枝分かれして見えるが、正本知識の分岐ではなく、Act memory 上の実行分岐である。

## 代表的な tools

### `get_visible_graph`

今キャンバスに見えているノード、エッジ、選択状態、active node を取得する。  
agent が現状把握をする最初の入口。

### `get_selected_nodes`

現在の選択文脈を読む。  
複数選択しているノードを次の探索の入力に使う。

### `open_node_detail`

指定ノードを右ペインに開く。  
agent が本文や action を確認する流れで使う。

### `submit_ask`

Ask フォーム相当の新規実行。  
空キャンバスからの探索や、通常の follow-up を始める。

### `create_selectable_nodes`

agent が複数の候補ノードを frontend に一時表示し、ユーザーにどれを採るか選ばせる。  
「候補を生成して終わり」ではなく、「候補を UI に出し、ユーザー選択を待つ」ための tool。

典型例:

* 調査の切り口を3案出す
* 次に深掘りするノード候補を並べる
* 要約案や整理案を複数提示して、人間に選ばせる

### `get_selection_group_result`

`create_selectable_nodes` の結果取得用。  
まだ選ばれていなければ `pending`、選ばれたら option/node 単位で結果を返す。

### `run_act_with_context`

起点ノード付きの派生実行。  
ノード内 action の `run_act` と同じ意味で使う。

### `report_stream_error`

stream 中の `terminal.error`、購読例外、想定外 event を dev console と計測へ送る。  
ユーザー向け表示とは別に、開発時の可観測性を確保するための tool。

## 重要な境界

### frontend state は正本ではない

tool で読めるのは「いま UI がどう見えているか」。  
知識正本や永続化契約の正本ではない。

### stream 契約は別にある

`submit_ask` や `run_act_with_context` は最終的に `RunAct` を叩く。  
実際の request/response 契約は `RunActRequest` / `RunActEvent` の仕様を正本にする。

### write tool でも commit はしない

frontend tool は、選択、表示、stream 開始、dev console 記録まではやる。  
永続知識への commit は別経路で扱う。

### 選択用ノードは一時物として扱う

`create_selectable_nodes` で出した候補ノードは、まず UI 上の選択支援である。  
この段階では knowledge 正本へ昇格しない。採用後に別フローで使う。

## Tool と Context Assembly の境界

ここは実装で混ざりやすいが、責務は分ける。

短く言うと:

* tool は材料を渡す
* assembly は材料を料理する

### tool がやること

tool は UI から観測できた事実やユーザー選択を返す。

例:

* いま選択されているノードID
* active node の本文
* visible graph の状態
* selection group でユーザーが選んだ option

ここでは、まだ model 入力向けの最適化をしない。

tool が返してよいもの:

* `node_id`
* `title`
* `content_md`
* `parent_id`
* `selected_option_ids`
* `selected_node_ids`

tool が返さない方がよいもの:

* relevance score
* token budget 後の圧縮本文
* prompt に貼る完成済み text
* retrieval 済みランキング結果

### assembly がやること

Context Assembly は、tool や request から来た材料を model に渡せる形へ変換する。

例:

1. explicit context node を読む
2. anchor 周辺を追加取得する
3. topic 全体から関連候補を検索する
4. ranking する
5. token budget に収める
6. model 用 bundle を作る

ここで初めて、`selected nodes` や `anchor` が「モデル入力にどう効くか」を決める。

### 実装フロー

実装は 4 段で分けると崩れにくい。

1. frontend tool が raw input を返す  
   例: `get_selected_nodes` が `node ids + raw markdown` を返す

2. agent が `RunAct` に渡す最小入力を作る  
   例: `context_node_ids`, `anchor`, `user_message`, 各種 config

3. backend / worker が assembly 専用 DTO を作る  
   `RunActRequest` をそのまま prompt builder に渡さない

4. assembly が retrieval / ranking / budgeting / bundle 化を行う  
   model には bundle だけを渡す

### DTO の分け方

実装では DTO を分けると責務が崩れにくい。

#### 1. Tool Result

UI 事実を表す。

```ts
type SelectedNodesToolResult = {
  nodes: Array<{
    nodeId: string
    title?: string
    contentMd?: string
    parentId?: string
  }>
  count: number
}
```

#### 2. RunActRequest

transport 契約を表す。

```ts
type RunActRequest = {
  topicId: string
  workspaceId: string
  userMessage: string
  anchorNodeIds?: string[]
  contextNodeIds?: string[]
}
```

#### 3. ContextAssemblyInput

worker 内部で使う入力を表す。

```ts
type ContextAssemblyInput = {
  topicId: string
  workspaceId: string
  userMessage: string
  anchorNodeIds: string[]
  explicitContextNodeIds: string[]
  uiSignals: {
    activeNodeId?: string
    selectedNodeIds?: string[]
  }
  profile: {
    useGrounding: boolean
    modelProfile: "flash" | "deep_research"
  }
}
```

#### 4. ContextBundle

model に渡す完成形を表す。

```ts
type ContextBundle = {
  intent: object
  pinnedContext: object[]
  retrievedContext: object[]
  compressedContextText: string
  citations: object[]
}
```

### 判断基準

どちらに置くか迷ったら、この基準で切る。

* UI 状態に依存するなら tool
* ユーザー選択の取得なら tool
* 正本データの retrieval なら assembly
* ranking / budget / compression なら assembly
* model prompt 最適化なら assembly

### アンチパターン

次は混線しやすいので避ける。

* tool が prompt 用の完成文章を返す
* frontend が ranking 済み context を作る
* agent が token budget を見て context を削る
* model prompt builder が `selectedNodeIds` の生配列を直接読む

これをやると、frontend tools が Context Assembly の代わりを始めてしまう。

### 最小実装案

最初はこれで十分。

* frontend tools は `node ids + raw markdown` を返す
* `submit_ask` / `run_act_with_context` は `context_node_ids` と `anchor_node_id` を送る
* worker 側で `ContextAssemblyInput` を作る
* assembly で本文取得・ranking・budgeting を行う
* model へは bundle だけを渡す

## MCP はどこに入るか

MCP は Context Assembly ではない。  
frontend tools を agent に公開するための接続方式である。

短く言うと:

* tool contract = 中身
* MCP = 配り方
* assembly = 使った後の文脈化

### MCP がやること

* tool discovery
* tool invocation
* transport/session 管理
* reconnect 時の再 discovery

### MCP がやらないこと

* retrieval
* ranking
* budgeting
* prompt 最適化
* model 用 context bundle の生成

つまり、MCP を入れても責務は変わらない。

* tool は raw input を返す
* assembly は model 用に料理する
* MCP はその tool を agent へ届ける

実装の置き場所としては、

* tool 自体の意味は `frontend-agent-tools.md`
* MCP 公開面は `frontend-agent-tools-mcp.md`
* MCP lifecycle は `frontend-agent-tool-mcp-lifecycle.md`

に分けるのが自然である。

## まず覚える運用ルール

* tool は意図単位で設計する
* read と write を分ける
* 派生 Act は anchored child run と考える
* frontend は agent の道具箱であり、知識正本ではない
* エラーはユーザー表示と dev console 表示を分けて扱う

## 正本参照

* `act/specs/contracts/frontend-agent-tools.md`
* `act/specs/contracts/frontend-agent-tools-mcp.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/frontend-stream-integration.md`
* `act/specs/behavior/frontend-agent-tool-mcp-lifecycle.md`
* `concept.md`
