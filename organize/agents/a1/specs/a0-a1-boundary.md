# A0 / A1 境界仕様

## A0 MediaInterpreter（本仕様の主対象）

A0は以下を担当する。

* 入力受理（`web_url | raw_html | gcs_object`）
* 軽抽出判定と重抽出回送（`extract.requested`）
* 抽出結果の保存（`inputs/{inputId}.extractedRef`）
* A1起動イベントの発火（`input.received`）

## A1 Atomizer（本仕様では契約のみ）

A1は以下を前提として消費する。

* Event: `input.received`
* Firestore: `inputs/{inputId}`
* 必須参照: `extractedRef`, `rawRef`, `traceId`, `origin`

## 入出力契約（A0 -> A1）

`input.received` payload（最小）:

```json
{
  "type": "input.received",
  "traceId": "t_xxx",
  "inputId": "in_xxx",
  "workspaceId": "ws_xxx"
}
```

A1が参照する Firestore:

```json
{
  "status": "extracted",
  "rawRef": { "gcsUri": "...", "generation": "...", "sha256": "...", "mimeType": "..." },
  "extractedRef": { "gcsUri": "...", "generation": "...", "sha256": "...", "mimeType": "text/plain" },
  "traceId": "t_xxx",
  "origin": { "sourceType": "...", "urlOrRef": "..." }
}
```
