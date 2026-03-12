# Frontend Agent Tools Acceptance

対象:

* `features/graph/components/CustomNode.tsx`
* `features/graph/components/GraphCanvas.tsx`
* selection group 用 store / hooks
* dev console / UI 通知連携

## 1. Selection Node 表示

* 選択用ノードが通常ノードと別 variant で描画される
* 選択用ノードが薄いアンバー背景 + 点線枠 + `Select` バッジを持つ
* `single` はラジオボタン風、`multiple` はチェックボックス風 UI で区別できる
* `selected` / `expired` / `cancelled` が視認できる

## 2. Selection Group ヘッダ

* selection group ヘッダに `title`, `instruction`, `status badge`, `選択数`, `Confirm/Clear/Cancel` が表示される
* `pending`, `selected`, `expired`, `cancelled` がヘッダだけで判別できる
* 文言が `Waiting for your choice`, `Selection confirmed`, `Selection expired`, `Selection cancelled` に整合する

## 3. 選択確定

* `single` は1クリックで確定する
* `multiple` は `Confirm` 前に複数候補を調整できる
* `multiple` の `Clear` で一括解除できる
* `expired` / `cancelled` / 確定済み状態では選択変更できない

## 4. 状態遷移と寿命

* 初期状態が `pending` で始まる
* 選択確定後に `selected` へ遷移する
* `expires_in_ms` 到達で `expired` へ自動遷移する
* `expired` で突然消えず、read-only のまま残る
* `Dismiss` または次の整理タイミングで片付けられる

## 5. 選択後の制御

* 選択確定だけでは新規 stream が自動開始されない
* frontend が選択結果を保持し、`get_selection_group_result` で取得できる前提と整合する
* 次アクションが agent 判断に委ねられる

## 6. 配置

* `anchor_node_id` ありでは selection group が起点ノード右側近傍に描画される
* `anchor_node_id` なしでは selection group が `choice lane` に描画される
* selection group が通常ノードの自動レイアウトと混ざらない
* 新しい pending group が既存 pending group と重なりにくい

## 7. 通常選択との分離

* 通常文脈選択と selection group 選択が store 上も UI 上も混同されない
* selection group 上の選択が `Ask` の通常文脈へ自動混入しない
* 通常選択と selection group 選択のハイライト意味が異なる

## 8. 待機表示

* agent 待機表示が selection group ヘッダで把握できる
* `Waiting for your input` と `Processing your selection` を使い分けられる
* キャンバス全体は操作可能なまま

## 9. エラー表示

* dev console 出力とユーザー向け画面通知の責務が分離されている
* `terminal.error` で UI は要約 message と retryable を表示できる
* `unexpected_event` / `reducer_failure` は console 主体で扱える
* 同一エラーで UI が過剰通知しない
