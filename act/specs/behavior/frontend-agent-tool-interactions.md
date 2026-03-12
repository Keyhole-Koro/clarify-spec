# Frontend Agent Tool Interactions

## 目的

frontend agent tools に対応する UI 挙動、状態遷移、表示ルールを固定し、agent と frontend が同じ interaction model で動作できるようにする。

## 対象

* `create_selectable_nodes`
* `get_selection_group_result`
* `report_stream_error`
* selection group の表示と選択確定

## 前提 / 参照

* `act/specs/contracts/frontend-agent-tools.md`
* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/frontend-stream-integration.md`
* `act/specs/quality/frontend-agent-tools-acceptance.md`

## Selection Group UI

### 見た目

* 選択用ノードは通常ノードと別 variant とする
* 通常ノードは白背景 + 通常枠、選択用ノードは薄いアンバー背景 + 点線枠 + `Select` バッジ
* `single` はラジオボタン風、`multiple` はチェックボックス風 UI をノード下部に表示する
* `selected` 状態では枠線強調、面変化、`Selected` バッジを表示する
* `expired` 状態ではグレーアウトして操作不能にする
* `cancelled` / 確定済み状態では read-only にする

### ヘッダ

* selection group ごとにヘッダを表示する
* ヘッダには `title`, `instruction`, `status badge`, `選択数`, `Confirm/Clear/Cancel` を表示する
* 状態表示はヘッダ主導とする
* 表示文言:
  * `pending`: `Waiting for your choice`
  * `selected`: `Selection confirmed`
  * `expired`: `Selection expired`
  * `cancelled`: `Selection cancelled`

## Selection 操作

* `single` はノードクリックで即時確定する
* `single` は別候補クリック時に選択を置き換える
* `multiple` はノードクリックで toggle する
* `multiple` は selection group 単位の `Confirm` で確定する
* `multiple` には `Clear` 操作を用意する
* `expired` / `cancelled` 状態では選択操作を受け付けない

## 状態遷移

* 初期状態は `pending`
* ユーザーの選択確定で `selected` へ遷移する
* `expires_in_ms` 到達時は `pending -> expired` へ自動遷移する
* `cancel` 操作時は `cancelled` へ遷移する
* `expired` / `cancelled` / `selected` 後は read-only とする
* 期限切れ後に自動削除はしない
* `expired` では `Confirm` / `Clear` / 選択変更を無効化する
* 後から `Dismiss` または次の整理タイミングで片付けられるようにする

## 選択後の制御

* ユーザー選択確定後に frontend が自動で `run_act_with_context` を起動しない
* selection group は `selected` に遷移し、frontend は選択結果を保持する
* agent は `get_selection_group_result` で結果を取得する
* 次アクションは agent が判断する
  * `run_act_with_context`
  * `open_node_detail`
  * 再度の候補提示
  * 何もしない

## 配置ルール

* `anchor_node_id` がある場合は、起点ノードの右側を基本に selection group を縦並び配置する
* `anchor_node_id` がない場合は、通常ノードレイアウトと分離した `choice lane` に配置する
* 候補ノードは selection group 単位でまとまって表示し、通常ノードの自動レイアウトに混ぜない
* 起点ノードがある場合は、起点と selection group の関係が分かる補助線または近接配置を維持する
* 新しい selection group は既存 pending group と重ならないよう縦方向にずらして配置する

## 通常選択との分離

* `selectedNodeIds` は通常の文脈選択専用とする
* selection group の選択状態は別 state として管理する
* selection group 上の選択は `Ask` の通常文脈へ自動混入させない
* 通常ノードの選択ハイライトと selection group の選択ハイライトは意味を分けて表示する

## 待機表示

* 待機表示は画面全体の blocking UI ではなく、selection group ヘッダ内の軽い状態表示とする
* ユーザー入力待ちは `Waiting for your input` を表示する
* 選択確定後に agent が次判断中であれば `Processing your selection` を表示できるようにする
* 期限切れ時は `Selection expired` を表示する
* 待機表示のためにキャンバス全体の操作は塞がない

## エラー表示

* dev console は開発者向け診断面とし、表示可能なエラー詳細を最大限出す
* 画面通知はユーザー向け案内面とし、行動に必要な最小情報のみ出す
* `terminal.error` は UI に要約 message と retryable を出し、console には `trace_id`, `request_id`, `stage`, `retryable`, `message`, 安全な raw event を出す
* `unexpected_event` と `reducer_failure` は console 主体で扱い、UI は必要時のみ汎用エラーを出す
* 同一エラーのトースト連打は抑制する

## MUST / 制約

* selection group は知識正本ではなく、一時的な UI interaction として扱う
* selection group の状態変化は通常ノードの stream 描画を壊してはならない
* 自動実行よりも、選択結果を agent へ返して次判断に委ねることを優先する
* choice lane は通常グラフの構造理解を阻害しない位置に保つ

## 完了条件（DoD）

* agent tools に対応する UI 挙動が一意に説明できる
* selection group の見た目、操作、寿命、配置、待機表示が固定されている
* dev console とユーザー向け画面通知の責務が分離されている
