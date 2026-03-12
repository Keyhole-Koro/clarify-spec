# A0 MediaInterpreterAgent 仕様

## 1. 責務

* `media.received` を受け取り、テキスト化して `input.received` を発火
* 詳細仕様: `organize/agents/a0/specs/ingest-extract-spec-v0.3.md`

## 2. I/O

* Input: `media.received`
* Output: `inputs/{inputId}`, `mind/inputs/{inputId}.md`
* Emit: `input.received`

## 3. Idempotency / 競合対策

* ledger: `type:media.received/inputId:{inputId}`
* 同一 `sha256` かつ status=processed はスキップ

## 4. 未決定事項

* 対応メディア形式（audio/pdf/image）
* OCR/ASRモデル選定
* 最大ファイルサイズ
