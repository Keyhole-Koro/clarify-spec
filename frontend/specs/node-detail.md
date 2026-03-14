# Node Detail Spec

目的: node の詳細を、A7 の要約と canonical Markdown を分離して表示する。

## 1. Source of Truth

* `frontend/frontend-spec.md`
* `organize/agents/a7/README.md`
* `organize/specs/pipeline/agents.md`

## 2. 役割

`node-detail` は次の 2 層を扱う。

* A7 由来の summary layer
* canonical Markdown body

A7 の `detailHtml` は本文の代替ではなく、本文の前置きサマリーとする。

## 3. レイアウト

右ペイン内を次の順で表示する。

1. node header
* title
* kind
* state badge があればその表示

2. summary card
* `contextSummary`
* `detailHtml`

3. canonical body
* `contentMd`

4. evidence / related
* evidence refs
* 関連 node
* actions

## 4. `contextSummary` の扱い

用途:

* list/card/graph 上の短表示
* node detail 上部の短説明

要件:

* 1-2 文で読める
* title の繰り返しだけで終わらない
* relation を最低 1 つ含む

## 5. `detailHtml` の扱い

用途:

* node detail の summary card

要件:

* 根拠・重要点・関係性を短く読める
* 見出し・箇条書き・軽い強調を許容
* 本文の代わりに長文化しない

禁止:

* `detailHtml` を canonical body として扱うこと
* `detailHtml` のみで node detail を完結させること

## 6. Markdown body の扱い

`contentMd` は canonical 本文として扱う。

要件:

* Markdown renderer で表示する
* `node://` リンクを選択同期に使う
* sanitize を前提とする

未確定:

* MVP で編集可にするか
* 目次や折りたたみを入れるか

## 7. データ欠損時の表示

### `detailHtml` がない

* `contextSummary` と Markdown body のみ表示する

### `contentMd` がない

* `detailHtml` を summary として表示しつつ、「本文未生成」扱いにする

### evidence がない

* evidence セクション自体を隠してよい

## 8. Graph との役割分担

Graph 上では full body を出さない。

Graph で使ってよいもの:

* title
* kind
* `contextSummary`

右ペインにだけ出すもの:

* `detailHtml`
* Markdown body
* evidence list

## 9. actions の配置

actions は node header か summary card の下に置く。

MUST:

* 本文中に actions を混ぜない
* destructive action は summary より下に置く

## 10. 未決事項

人間判断が必要:

* Markdown 編集を MVP に含めるか
* evidence を専用 UI にするか Markdown 内表現に寄せるか
* related nodes を右ペインで見せるか、Graph 遷移だけにするか
