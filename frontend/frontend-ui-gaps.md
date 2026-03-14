# Frontend UI Spec Gaps (MVP)

この文書は、既存仕様から見て「UI記述が不足している点」を列挙する。
対象は `frontend/frontend-spec.md`、`act/specs/behavior/*`、`act/specs/usecases/*`、`organize/specs/*`。

## 1. 画面レイアウトの確定値不足

不足:

* 左ペイン/キャンバス/右ペインの幅ルールが固定されていない
* 右ペインの表示方式（固定表示 or スライドイン）が最終確定していない
* モバイル時のレイアウト崩し方（非表示/ドロワー化）が未定義

決めること:

* `>=1280px`, `>=768px`, `<768px` の3段階レイアウト
* 右ペインの既定幅と最小幅

## 2. Askフォームの状態定義不足

不足:

* `idle/running/error` 時のボタン状態（disabled, label）が未定義
* 連投時の扱い（前リクエスト中の再送可否）が未定義
* 入力文字数上限超過時のUI挙動が未定義

決めること:

* 送信中のUI（スピナー、キャンセル可否）
* Enter送信/Shift+Enter改行のキー仕様

## 3. Thinkthrough UIの具体形不足

不足:

* thought表示位置（右ペイン/下部パネル/ノード内）が未確定
* thought ON/OFF トグルの配置が未定義
* thoughtを保存するか、セッション限定かが未定義

決めること:

* thought表示コンポーネントと折りたたみ挙動
* answerとの見た目差分（色/ラベル/区切り）

## 4. ノード選択・操作の詳細不足

不足:

* single click / double click / drag の優先順位が未定義
* 複数選択中のコンテキストメニュー仕様がない
* 選択解除のルール（空白クリック/Esc）が未定義

決めること:

* `onNodeClick` と `onNodeDoubleClick` の役割分担
* 選択中バッジ・件数表示の有無

## 5. アクションボタンUI仕様不足

不足:

* `actions[]` の表示上限、折り返し、overflow時の扱いが未定義
* `run_act` 実行中に同一ノードアクションを再押下できるか未定義

決めること:

* ボタン表示個数上限（例: 3件 + More）
* 実行中のローディング表示と再押下防止

## 6. エラー表示・復帰導線不足

不足:

* エラーの表示レベル（toastのみ/固定バナー併用）が未定義
* `UNAUTHENTICATED` / `UNAVAILABLE` / `DEADLINE_EXCEEDED` のUI出し分けが未定義
* リトライ導線の位置（Askフォーム内/トースト内）が未定義

決めること:

* エラーコードごとの文言テンプレート
* 再試行ボタンの共通配置

## 7. 認証UI（Google限定）の不足

不足:

* 未ログイン時の画面（全画面ガード/軽量バナー）が未定義
* Googleログイン失敗時のUIと復帰導線が未定義
* ログイン後のユーザー表示（アイコン/メニュー）が未定義

決めること:

* ログイン必須画面の最小UI
* サインアウト導線の位置

## 8. Mockモード表示不足

不足:

* Mock/Real の現在モード表示が未定義
* Mockデータ再生成（seed変更）UIが未定義

決めること:

* ヘッダーに `MOCK` バッジを出すか
* seed再生成ボタンの有無

## 9. MarkdownPaneの編集境界不足

不足:

* 閲覧専用か簡易編集可かの最終判定が未確定
* 長文時の折りたたみ/目次表示が未定義
* `node://` リンククリック時の遷移演出が未定義

決めること:

* MVPで編集機能を入れるか（入れないなら明示）
* 内部リンククリック時の選択同期ルール

## 10. アクセシビリティ最低基準不足

不足:

* フォーカス順とキーボード操作一覧が未定義
* 色のみで状態識別していないかの基準が未定義

決めること:

* 最低ショートカット（送信、選択解除、ペイン閉じる）
* ARIAラベル付与対象

## 11. ローディング表現不足

不足:

* 初回読み込み、stream中、コミット中のローディングUI差分が未定義
* skeletonを使う画面領域が未定義

決めること:

* ローディング種別ごとのUI部品を固定
* 表示遅延閾値（例: 300ms超で表示）を決める

## 12. 運用向けUI計測不足

不足:

* UIイベント計測項目（送信、失敗、再試行、選択件数）が未定義
* `traceId` の画面確認方法が未定義

決めること:

* Dev向けデバッグ表示トグル
* 最低限の計測イベント名

## 13. アイコン運用ルール不足

不足:

* Lucide/Tablerのどちらを優先するか画面単位で未定義
* アイコンサイズと線幅の統一基準が未定義
* 意味カテゴリ（状態/操作/警告）ごとの選定ルールが未定義

決めること:

* 画面ごとの icon set 優先順位（基本 Lucide、補完でTabler など）
* サイズトークン（16/20/24）と stroke 幅
* 参照先URL一覧を実装者向けテンプレートへ固定記載

## 14. Patch責務過密リスク

不足:

* `applyPatch` に stream制御/状態更新/描画変換/UI副作用が混在しやすい
* 不具合発生時に再現経路の切り分けが難しい

決めること:

* Stream Adapter / Patch Reducer / Graph Projection / UI Store の4分割を固定
* 各層の入出力境界（副作用可否）を明文化

## 15. TopicResolver 判定可視化不足

不足:

* `resolvedTopicId`, `resolutionMode`, `resolutionConfidence`, `resolutionReason` を UI のどこに出すか未定義
* 既存 topic attach と新規 topic 作成の差がユーザーに見えない
* 誤 attach の監査導線がない

決めること:

* input 詳細または activity panel に routing result を表示する
* `attach_existing | create_new` をバッジで区別する
* confidence と reason の表示粒度を固定する

## 16. Organize 更新タイムライン不足

不足:

* `draft.updated -> bundle.created -> outline.updated` を1本の更新フローとして見せる画面がない
* A2/A3b/A3/A6 の成果物が別々に存在し、今回の入力が何を変えたか追いにくい
* bundle description をどの画面で使うか未定義

決めること:

* topic 単位の activity timeline を追加する
* 1入力ごとに draft diff, bundle preview, outline反映結果を束ねて表示する
* A6 の `descRef` を timeline 内 preview として使う

## 17. Node Detail 二層表示不足

不足:

* A7 の `contextSummary` と `detailHtml` を UI でどう使い分けるか未定義
* Graph 上の簡潔表示と右ペイン詳細表示の境界が曖昧
* MarkdownPane 中心構成と A7 HTML 出力の接続が未定義

決めること:

* 右ペイン上部に短要約カード、下部に根拠付き詳細を置く
* Graph/mini panel では `contextSummary`、詳細閲覧では `detailHtml` を使う
* Markdown 本文との優先順位を固定する

## 18. `organizeOps` Review Inbox 不足

不足:

* `organizeOps/{opId}` の消費者が UI 上で未定義
* `planned / approved / applied / dismissed` の操作面がない
* 通常の閲覧 UI と運用判断 UI が分離されていない

決めること:

* review inbox を topic 詳細とは別導線で持つ
* op state ごとの表示列とフィルタを定義する
* approve/dismiss/apply の権限と操作確認 UI を定義する

## 19. Inspector Local Web UI 不足

不足:

* phase runner / inspector は CLI 前提で、A3 diff や emit preview の視認性が低い
* preview object をブラウザで比較する仕様がない
* 開発者向け UI と本番 UI の境界が未定義

決めること:

* `/inspector` のような local-only web UI を持つか決める
* Firestore/GCS/event preview の並列表現を定義する
* A3 の node/edge diff と `atom.reissued` 候補を可視化する

## 20. Upload処理進捗表示不足

不足:

* フロントから upload した input が今どの phase にいるか見えない
* `media.received -> input.received -> atom.created -> topic.resolved -> draft.updated -> bundle.created -> outline.updated` をユーザー向け段階にどう写像するか未定義
* retry 中と失敗確定の UI 出し分けが未定義

決めること:

* upload status を `uploaded / extracting / atomizing / resolving_topic / updating_draft / completed / failed` のような段階へ固定する
* header の簡易 progress と activity panel の詳細 progress の役割分担を決める
* `completed` 時に attach/create_new 結果と反映先 topic をどう出すか決める
* 長時間停滞時の `processing delayed` 表示と `failed` への遷移条件を決める

## 優先度（MVP実装前に決める順）

1. レイアウト確定（1）
2. Askフォーム状態（2）
3. Upload処理進捗表示（20）
4. TopicResolver 判定可視化（15）
5. Organize 更新タイムライン（16）
6. エラー復帰導線（6）
7. 認証UI（7）
8. Node Detail 二層表示（17）
9. Thinkthrough表示（3）
10. ノード操作詳細（4, 5）
11. `organizeOps` Review Inbox（18）
12. Mock表示（8）
13. MarkdownPane境界（9）
14. Inspector Local Web UI（19）
15. A11y/ローディング/計測（10, 11, 12）
16. アイコン運用（13）
17. Patch責務分離（14）
