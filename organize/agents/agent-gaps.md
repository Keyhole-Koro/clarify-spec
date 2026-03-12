# Organize Agent Gaps

## 目的

各 Agent の README と pipeline 正本を見た時点で、実装に進む前に追加で固定すべき仕様不足を一覧化する。

## 全体ギャップ

### 1. 入出力 schema の不足

多くの Agent で、イベント payload と Firestore/GCS 出力のフィールド定義が不足している。

不足例:

* 入力イベントの必須/任意フィールド
* 出力ドキュメントの JSON schema
* versioning ルール
* max size / 分割方針

### 2. 判定アルゴリズムの不足

責務は書かれているが、「どう判定するか」が未固定な Agent が多い。

不足例:

* merge / create / reject 基準
* topic routing 基準
* bundle 分割基準
* index feature 算出基準
* rebalance 発火基準

### 3. 異常系の不足

retry / skip / DLQ の条件が Agent 単位では十分に落ちていない。

不足例:

* retryable / non-retryable の分類
* stale event の扱い
* partial failure の戻し方
* `atom.reissued` の厳密条件

### 4. 観測性の不足

trace / log / metric の必須項目が未定義。

不足例:

* span 名
* counter / histogram 名
* ledger / lease / CAS miss のログ属性

### 5. テスト受け入れ条件の不足

README レベルでは DoD と fixture 例が不足している。

不足例:

* idempotency テスト
* lease 競合テスト
* stale event test
* output schema snapshot test

## Agent 別ギャップ

## A0 MediaInterpreter

対象: [a0/README.md](/home/unix/clarify-spec/organize/agents/a0/README.md)

不足しているもの:

* 対応メディア形式ごとの処理分岐
* light / heavy extract の判定閾値
* `inputs/{inputId}` の最終 schema 要約
* MIME 判定ルール
* `status/error.stage` の状態遷移詳細
* 最大ファイルサイズとタイムアウト

## A1 Atomizer

対象: [a1/README.md](/home/unix/clarify-spec/organize/agents/a1/README.md)

不足しているもの:

* `atoms/{atomId}` schema
* atom の粒度ルール
* title / claim / kind / confidence / source span の定義
* deterministic `atomId` の生成式
* `atom.created` payload の上限
* 再抽出時の上書き/追記ルール

## A2 Router

対象: [a2/README.md](/home/unix/clarify-spec/organize/agents/a2/README.md)

不足しているもの:

* atom をどの topic に入れるかの routing ルール
* draft 追記位置の規則
* draft 本文フォーマット
* `draft.updated` payload の詳細 schema
* 長大 draft の圧縮 / rotation ルール
* topic 不確定時の fallback

## A3b Bundler

対象: [a3b/README.md](/home/unix/clarify-spec/organize/agents/a3b/README.md)

不足しているもの:

* `PipelineBundle` schema
* bundle と distillation の概念整理
* bundle 分割アルゴリズム
* `topic.schema_updated` 発火条件
* schema 更新の minor / major 区分
* schema proposal の JSON shape
* A3 / A6 に渡す payload の最適化方針

優先度:

* 最優先

## A3 Cleaner

対象: [a3/README.md](/home/unix/clarify-spec/organize/agents/a3/README.md)

不足しているもの:

* entity resolution の判断特徴量
* merge / rewrite / reject 基準
* edge upsert key 設計
* outline 再生成アルゴリズム
* `atom.reissued` の厳密条件
* changed node の選定基準
* schema 不整合時の degrade ルール

優先度:

* 最優先

## A4 Indexer

対象: [a4/README.md](/home/unix/clarify-spec/organize/agents/a4/README.md)

不足しているもの:

* `index_items/*` schema
* keyword / vector / hybrid のどれを正本にするか
* importance / freshness / confidence の算出式
* 全量再計算か差分更新か
* `latestMapVersion` 更新条件
* map markdown の構造

## A5 Balancer

対象: [a5/README.md](/home/unix/clarify-spec/organize/agents/a5/README.md)

不足しているもの:

* `topic.metrics.updated` payload schema
* `organizeOps/{opId}` schema
* metrics 定義
* rebalance 発火条件
* 自動是正の範囲
* 人手レビュー閾値
* cron / event driven の最終決定

## A6 BundleDescription

対象: [a6/README.md](/home/unix/clarify-spec/organize/agents/a6/README.md)

不足しているもの:

* description HTML の用途固定
* HTML schema / template contract
* `descRef` versioning ルール
* 再生成条件
* CSS / asset 配信方針
* UI 表示との関係

## A7 NodeRollup

対象: [a7/README.md](/home/unix/clarify-spec/organize/agents/a7/README.md)

不足しているもの:

* rollup schema
* `context_summary_ref` の長さ / 粒度制約
* rollup 再生成トリガー条件
* `rollupWatermark` の意味
* 親子 node 集約ルール
* Act 注入用 summary と UI 詳細本文の分離契約

## 優先順位

先に詰めるべき順:

1. A3b の `PipelineBundle` schema
2. A3 の merge / entity resolution / reissue 基準
3. A1 の atom schema と分割ルール
4. A4 の index schema
5. A7 の rollup schema

## メモ

このファイルはギャップ一覧であり、正本仕様ではない。  
正本へ反映する際は `organize/specs/pipeline/agents.md` と各 Agent `README.md` に戻して固定する。
