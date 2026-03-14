# AGENTS.md

このファイルは、このリポジトリで作業するエージェント向けの実務ルールを定義する。
目的は、仕様の参照先を迷わないこと、未定義の実装判断を勝手に埋めないこと、環境変数の扱いを曖昧にしないこと。

## 1. 基本方針

* 仕様の正本はコードではなく Markdown にある
* 実装・修正の前に、対象領域の Source of Truth を読む
* 仕様に明記されていない挙動は推測で固定しない
* 一時しのぎの fallback で動かさない

## 2. 仕様ルーティング

作業対象ごとに、まず読むべき仕様を固定する。

### 2.1 全体把握

1. `IMPLEMENTATION-INDEX.md`
2. `concept.md`
3. `context/README.md`
4. `organize/README.md`
5. `frontend/frontend-spec.md`

### 2.2 Frontend

Source of Truth:

1. `frontend/frontend-spec.md`
2. `frontend/frontend-ui-gaps.md`
3. `act/specs/behavior/frontend-stream-integration.md`
4. `act/specs/behavior/frontend-canvas-phases.md`
5. `act/specs/quality/frontend-stream-acceptance.md`

追加で参照:

* upload 進捗表示: `firestore/schema.md`, `organize/specs/pipeline/agents.md`
* 認証/セッション: `act/act-api/specs/security/session-and-auth-boundary.md`

### 2.3 Organize

Source of Truth:

1. `organize/specs/README.md`
2. `organize/specs/pipeline/summary.md`
3. `organize/specs/pipeline/spec.md`
4. `organize/specs/pipeline/core.md`
5. `organize/specs/pipeline/agents.md`
6. `organize/specs/pipeline/ops.md`
7. `organize/specs/pipeline/topic-resolution.md`
8. `firestore/schema.md`

Agent 個別仕様:

* `organize/agents/README.md`
* `organize/agents/a0/README.md`
* `organize/agents/a1/README.md`
* `organize/agents/a2/README.md`
* `organize/agents/a3b/README.md`
* `organize/agents/a3/README.md`
* `organize/agents/a4/README.md`
* `organize/agents/a5/README.md`
* `organize/agents/a6/README.md`
* `organize/agents/a7/README.md`

精度改善チケット:

* `organize/specs/pipeline/precision-improvement-tickets.md`

### 2.4 Firestore / データモデル

Source of Truth:

1. `firestore/schema.md`
2. `firestore/indexes.md`
3. `context/model/topic-model.md`

### 2.5 Act

Source of Truth:

1. `act/specs/README.md`
2. `act/specs/contracts/rpc-connect-schema.md`
3. `act/specs/behavior/act-flow.md`
4. `act/specs/behavior/frontend-stream-integration.md`
5. `act/act-api/specs/runtime/runact-implementation.md`
6. `act/act-api/specs/security/session-and-auth-boundary.md`

## 3. 環境変数ルール

環境変数に fallback を設けない。

MUST:

* 必須環境変数が無い場合は起動時または初期化時に即 error を出す
* `undefined` / 空文字 / ダミー既定値で処理を継続しない
* `dev だから local default を入れる` のような暗黙 fallback を実装しない
* 複数候補から順に拾う fallback も禁止する

実装原則:

* `env.ts` や設定初期化層で必須値を一括検証する
* missing 時は「どの環境変数が足りないか」が分かる error message を返す
* optional な環境変数だけを明示的に optional とする

例:

* 良い: `NEXT_PUBLIC_RPC_BASE_URL is required`
* 悪い: `process.env.NEXT_PUBLIC_RPC_BASE_URL ?? "http://localhost:8080"`

## 4. 仕様未定義時の判断ルール

仕様にない箇所を見つけたら、人間の判断を仰ぐ。

MUST:

* 未定義の仕様を勝手に埋めて実装しない
* 複数の解釈が成立する場合は、そのまま作業を進めず確認する
* 既存仕様同士が衝突している場合は、人間にエスカレーションする

エスカレーションが必要な例:

* event payload に必要な field が仕様にない
* UI の状態遷移は記述されているが timeout 条件がない
* Firestore path はあるが責務主体が書かれていない
* agent README と pipeline spec で挙動が食い違う
* schema の物理パスはあるが lifecycle / CAS 条件がない

確認せず進めてよい例:

* typo 修正
* 明らかな参照パス修正
* 仕様に既に書かれている内容の反映

## 5. 実装前チェック

実装前に次を確認する。

1. 対象の Source of Truth を読んだか
2. 変更対象の仕様間で矛盾がないか
3. 必須環境変数が定義されているか
4. 未定義箇所を推測で埋めようとしていないか

## 6. 仕様更新ルール

仕様を変更した場合、関連する正本も必要に応じて同時更新する。

例:

* Frontend の upload 表示を変えたら `frontend/frontend-spec.md` と `firestore/schema.md` と `organize/specs/pipeline/agents.md` を確認する
* Topic lifecycle を変えたら `organize/specs/pipeline/topic-resolution.md` と `firestore/schema.md` を確認する
* 新しい agent field を増やしたら `organize/specs/pipeline/agents.md` と agent 個別 README を確認する

## 7. 禁止事項

* 仕様未定義の fallback 実装
* 環境変数 missing を黙殺する実装
* Pub/Sub や内部イベントを frontend が直接読む前提の実装
* Source of Truth を読まずに局所ファイルだけで判断すること
