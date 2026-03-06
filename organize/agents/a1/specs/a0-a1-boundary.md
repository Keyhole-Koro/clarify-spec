# A0 / A1 境界仕様

## 目的

A0 MediaInterpreter と A1 Atomizer の接続契約を、コード片なしで固定する。

## スコープ / 非スコープ

* スコープ: A0出力とA1入力の最小契約
* 非スコープ: A1内部アルゴリズム

## A0 MediaInterpreter（主対象）

A0は以下を担当する。

* 入力受理（`web_url | raw_html | gcs_object`）
* 軽抽出判定と重抽出回送（`extract.requested`）
* 抽出結果の保存（`inputs/{inputId}.extractedRef`）
* A1起動イベントの発火（`input.received`）

## A1 Atomizer（契約のみ）

A1は以下を前提として消費する。

* Event type: `input.received`
* Firestore参照: `inputs/{inputId}`
* 必須参照: `extractedRef`, `rawRef`, `traceId`, `origin`

## 入出力契約（A0 -> A1）

### `input.received` の最小フィールド

| フィールド | 必須 | 説明 |
| --- | --- | --- |
| `type` | 必須 | `input.received` |
| `traceId` | 必須 | トレースキー |
| `inputId` | 必須 | 入力識別子 |
| `workspaceId` | 必須 | workspace境界 |

### `inputs/{inputId}` のA1必須参照項目

| フィールド | 説明 |
| --- | --- |
| `status` | `extracted` が前提 |
| `rawRef` | 元データ参照（gcsUri/generation/sha256/mimeType） |
| `extractedRef` | 抽出テキスト参照（gcsUri/generation/sha256/mimeType） |
| `traceId` | ログ相互参照 |
| `origin` | 入力起点情報 |

## 受け入れ条件（DoD）

* A1がA0出力のみでAtom化開始できる
* A0/A1で必須フィールド名の解釈差分がない
