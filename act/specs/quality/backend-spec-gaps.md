# Backend Spec Gaps

この文書は、Act バックエンドと関連 frontend 仕様のうち、まだ未解決の項目だけを記録する。

## 1. ADK Worker 実装詳細ギャップ（未解決）

現状:

* `act-adk-worker` の runtime は主要フローを定義済みだが、実装時に差が出やすい境界が残っている

次アクション（要決定）:

* cancel伝播仕様
  * `act-api` からの中断を `GenerateWithModel` / 外部tool呼び出しへ即時伝播する方式
* stream順序保証
  * thought / answer / patch の flush順序と同一chunk内の並びルール
* model profile切替表
  * Gemini 3 Flash / grounding / deep research の選択条件を固定
* tool/grounding metadata 正規化
  * `RunActEvent` へ何を残し、何を圧縮/破棄するかを固定
* ADK workerテスト仕様
  * 最小ゴールデンケース（intent, budget超過, error mapping, stream順序）を追加

## 2. Frontend 実装前に詰めるべき仕様境界（未解決）

現状:

* 共有仕様の大枠は固定されているが、文書間で解釈差が残る箇所と、UI実装時に判断が分かれやすい境界がある

次アクション（要決定）:

* terminal 挙動の整合
  * `done/error` をサーバ契約として厳密排他にする一方、フロントは二重終端受信時に無害化する実装を許容するかを固定
* Deep Research fallback の UI 通知
  * fallback 発生をユーザーに表示するか、dev console/log のみで追跡するかを固定
* thought 表示 UX
  * thought 表示の初期値、実行中のみ表示するか、完了後も保持するか、再実行時のクリア条件を固定
* grounding / tool metadata の UI 投影
  * 参照リンク、根拠、diagnostics をどの UI 領域に出すか、未対応時は store 保持のみにするかを固定
