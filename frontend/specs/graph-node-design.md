# Graph Node Design Spec

目的: Graph 上の node を、短時間で意味が分かり、右ペイン詳細へ自然に遷移できる単位として設計する。

## 1. Source of Truth

* `frontend/frontend-spec.md`
* `frontend/specs/node-detail.md`
* `organize/agents/a7/README.md`

## 2. 基本方針

Graph node は「全文を読む場所」ではない。

役割:

* node の種類を瞬時に判別させる
* title を主表示する
* `contextSummary` を短く補助表示する
* 右ペインの `node-detail` へ遷移させる

Graph node に載せすぎてよい情報ではない:

* Markdown 本文全文
* 長い evidence
* `detailHtml` 全文
* 多数の action

## 3. ノードの3サイズ

### `compact`

用途:

* 密度優先の全体俯瞰

表示:

* `kind` badge
* `title` 1 行

非表示:

* summary
* meta counters

### `default`

用途:

* 通常の topic 閲覧

表示:

* `kind` badge
* `title` 最大 2 行
* `contextSummary` 最大 2 行
* meta row 1 行

### `focus`

用途:

* selected node または spotlight node

表示:

* `kind` badge
* `title` 最大 2 行
* `contextSummary` 最大 3 行
* meta row
* 更新バッジまたは state badge 1 個まで

## 4. ノードの中身

表示順は次で固定する。

1. Top row
* `kind` badge
* optional state badge

2. Title
* node title

3. Summary
* `contextSummary`

4. Meta row
* relation count または evidence count
* 必要なら updated indicator

## 5. バッジ

### `kind` badge

必須:

* すべての node に表示する

ルール:

* 色だけでなくラベル文字も持つ
* badge 文言は短くする

例:

* `concept`
* `agent`
* `method`
* `benchmark`
* `person`

### state badge

用途:

* `new`
* `updated`
* `review`

制約:

* 同時表示は 1 個まで
* `kind` badge より弱い視覚強度にする

## 6. テキスト量

`title`

* 最大 2 行
* 溢れは省略

`contextSummary`

* `default`: 最大 2 行
* `focus`: 最大 3 行
* sentence を途中で切りすぎない

meta row

* 1 行固定
* count と短ラベルのみ

## 7. 状態

最低限持つ状態:

* `default`
* `hover`
* `selected`
* `dimmed`
* `newly_updated`

### `default`

* 通常表示

### `hover`

* border または shadow をわずかに強める
* summary や badge の配置は変えない

### `selected`

* border を最も強くする
* canvas 上で一目で分かること
* 右ペインの `node-detail` と同期する

### `dimmed`

* 非選択周辺のノードを少し落とす
* ただし読めなくしない

### `newly_updated`

* outline 更新直後のノード
* 一時的な badge または背景 tint で示す

## 8. 色と視覚方向

MUST:

* 白背景に薄グレー box のような無個性なカードにしない
* topic map として空間の奥行きが出るよう、背景・影・境界に差をつける
* 色だけで状態を表現しない

推奨:

* base: warm neutral
* accent: 緑または青緑系
* warning: 黄土・琥珀系

禁止:

* `selected` を色変化だけで表現すること
* `kind` ごとに派手すぎる多色運用

## 9. クリック挙動

single click:

* node を選択する
* 右ペインを `node-detail` に切り替える

double click:

* MVP では未使用でよい
* single click と競合するなら実装しない

background click:

* 選択解除

drag:

* layout mode でのみ許可してよい
* click との優先順位を明示する

## 10. アクション配置

Graph node 上の action は最小にする。

MVP:

* node 内常設 action は置かなくてよい
* hover 時に 1 個だけ quick action を出してよい
* 主要 action は右ペインに寄せる

禁止:

* 3 個以上の常設 action を node 上に置く
* 本文と action を同じ視覚階層に置く

## 11. モバイル

モバイルでは `compact` を基本にする。

ルール:

* summary は省略してよい
* selected node は right drawer 側で読む
* node 密度より tap しやすさを優先する

## 12. 未決事項

人間判断が必要:

* `meta row` に何を出すか
* `newly_updated` を何秒表示するか
* hover quick action を入れるか完全に右ペイン寄せにするか
