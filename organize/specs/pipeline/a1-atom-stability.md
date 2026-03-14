# A1 Atom Stability 仕様

Version: 1.0

目的: A1 Atomizer の atom 粒度、`kind`、`confidence` の再現性を上げる。

## 1. 基本方針

A1 は 2 段階で動く。

1. claim boundary を決定論的に切る
2. Gemini は各 claim 候補を正規化し、`kind` と `confidence` を付与する

MUST:

* Gemini に claim boundary 決定を丸投げしない
* `claim_count` の揺れを先に抑える
* LLM 失敗時も boundary 候補は保持する

## 2. Stage 1: Deterministic Claim Boundary

入力:

* `extractedText`

出力:

* `claimCandidates[]`

処理:

* 文分割を先に行う
* 接続詞や並列構造で複数主張を含む文を追加分割する
* 箇条書き・表セル・見出し直下の文を独立候補として扱う
* 引用符や括弧内補足だけでは独立 claim を作らない

推奨 split signal:

* `。`, `.`, `;`
* 箇条書き marker
* `しかし`, `一方で`, `また`, `さらに`, `because`, `therefore` などの discourse marker
* `A は B であり、C は D である` のような並列構文

MUST NOT:

* 境界決定時に推論的な意味補完をしない
* 1 candidate に topic が異なる主張を意図的に残さない

## 3. Stage 2: Gemini Normalization

各 `claimCandidate` ごとに Gemini へ渡す。

入力:

* `claimCandidate`
* `sourceSpan`
* `neighborCandidates`（前後 1 件）

出力（structured）:

* `title`
* `claim`
* `kind`
* `entities`
* `confidence`
* `reject`
* `rejectReason`

Gemini の責務:

* claim を短く正規化する
* 文脈補完を最小限に留める
* `kind` を 1 つだけ選ぶ
* `confidence` をレンジに従って返す
* atom として不適切なら `reject=true`

Gemini の非責務:

* candidate 数を増減させること
* 近接 candidate の merge
* source span の再解釈

## 4. Reject 条件

以下は `reject=true` 候補。

* 意味のある主張が成立していない断片
* 見出しだけで本文がないもの
* ノイズ、OCR 崩れ、文字化け
* 完全な重複 candidate

`reject=true` の candidate は atom 化しないが、監査用に件数を記録する。

## 5. kind 判定ルールの固定

`kind` は次の 5 値に固定する。

* `fact`
* `definition`
* `relation`
* `opinion`
* `temporal`

優先順位:

1. 時系列依存が本質なら `temporal`
2. 関係記述が主なら `relation`
3. 定義文なら `definition`
4. 主観評価なら `opinion`
5. それ以外で検証可能なら `fact`

## 6. confidence の固定レンジ

返してよい値は連続値ではなく、次の bucket に丸める。

* `0.95`
* `0.8`
* `0.6`
* `0.4`
* `0.2`

これにより微妙な数値揺れを減らす。

## 7. Atom ID の作り方

`atomId` は次で決定する。

* `sha256(topicId + sourceInputId + candidateIndex + normalizedClaim)`

`candidateIndex` は Stage 1 の順序を使う。

## 8. テスト仕様

最低限の golden fixture を持つ。

* 単文 1 claim
* 1 文 2 claim
* 箇条書き 3 項目
* OCR 崩れを含む入力
* 定義 / 関係 / 時系列 / 意見 の代表ケース

評価項目:

* candidate 数
* reject 数
* final atom 数
* `kind` の一致
* `confidence` bucket の一致

## 9. A3 との境界

A1 は「atom 粒度の安定化」までを担当する。

* A1 は node 化しない
* A1 は claim 間矛盾を解決しない
* A3 が merge / create / reject を最終判断する

## 10. 受け入れ条件

* 同一 input に対する atom 数の揺れが小さい
* 1 文多主張ケースで過剰結合しない
* `kind` の代表ケースが golden で固定できる
* LLM failure 時でも boundary candidate と reject 監査が残る
