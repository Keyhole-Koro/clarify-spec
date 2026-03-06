# Firestore Domain Docs

Firestore関連の正本をこのディレクトリに集約する。

## Files

* `firestore/schema.md`
  * Topic中心ER、物理パス、制約、transaction境界
* `firestore/indexes.md`
  * Topic中心の複合インデックス案
* `firestore/rules.md`
  * Security Rules方針（workspace/member/topic境界）

## Model Direction

* 知識正本キーは `topic_id`
* `draft` / `outline` / `nodes` / `actRuns` は topic 配下に配置
* 本文は GCS versioned ref、Firestoreは構造メタと参照ポインタ中心
