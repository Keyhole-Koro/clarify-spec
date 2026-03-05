# A1 AtomizerAgent 仕様

## 1. 責務

* `input.received` から Atom を抽出して `atom.created` を発火
* A0との境界契約: `organize/agents/a1/specs/a0-a1-boundary.md`

## 2. I/O

* Input: `input.received`
* Output: `atoms/{atomId}`, `mind/atoms/{atomId}.md`
* Emit: `atom.created`

## 3. Idempotency / 競合対策

* ledger: `type:input.received/inputId:{inputId}`
* 同一 input の二重生成防止（`sourceInputId` or deterministic atomId）

## 4. 未決定事項

* Atom分割ルール（長さ、粒度、タグ）
* span抽出精度要件
* 再抽出ポリシー
