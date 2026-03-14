# Review Inbox Spec

目的: `organizeOps` を通常閲覧 UI から分離し、運用判断の入口を提供する。

## 1. Source of Truth

* `frontend/frontend-spec.md`
* `organize/specs/pipeline/agents.md`
* `organize/specs/pipeline/ops.md`
* `organize/specs/pipeline/precision-improvement-tickets.md`

## 2. 役割

`review-inbox` は `organizeOps/{opId}` の一覧と状態を表示する。

この画面は次を担う。

* 提案 op の存在を知らせる
* op ごとの state を見せる
* trace と理由を追えるようにする

node detail や topic activity と責務を混ぜない。

## 3. 対象 op

想定対象:

* merge 提案
* split 提案
* archive 提案
* schema migration 提案
* rebuild / rebalance 提案

## 4. State

表示対象 state:

* `planned`
* `approved`
* `applied`
* `dismissed`

MVP 最小:

* read-only 表示でもよい
* ただし state badge は必須

## 5. 1カードの表示項目

必須:

* op title
* op type
* state
* reason
* createdAt
* traceId

推奨:

* target topic / node
* initiator
* requires human review フラグ

## 6. 並び順

既定:

* `planned`
* `approved`
* `applied`
* `dismissed`

同一 state 内:

* `createdAt desc`

## 7. フィルタ

MVP 最小:

* `all`
* `needs review`
* `approved`
* `history`

## 8. 権限

未確定でも最低限守ること:

* 誰でも approve/dismiss できる前提にしない
* write 操作は authz と紐づける

MVP read-only の場合:

* 権限未確定なら操作ボタンは出さない
* state と理由だけを表示する

## 9. 詳細表示

カードを開いたときに出してよいもの:

* full reason
* related topic / node
* source metrics
* traceId
* suggested action

通常一覧で出しすぎないもの:

* raw payload 全文
* internal debug object

## 10. topic activity との関係

`review-inbox` は input 単位ではなく op 単位で見る画面。

`topic-activity` との違い:

* `topic-activity`: 1 input の処理結果を見る
* `review-inbox`: 人間や system が消費すべき op を見る

## 11. 未決事項

人間判断が必要:

* MVP で approve / dismiss を UI から可能にするか
* state 更新主体を frontend に持たせるか backend workflow に限定するか
* op type ごとの詳細差分表示を別画面にするか modal にするか
