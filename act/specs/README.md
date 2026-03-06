# Act Specs Guide

Act仕様は以下の6層で管理する。

## Layers

* `overview/`: 全体像、読み順、DoD
* `context/`: Context Assembly と Bundle 契約（Act実行前の文脈層）
* `contracts/`: RPC/イベント/外部APIの入出力契約（破壊的変更管理対象）
* `behavior/`: フロー、状態遷移、実行ルール
* `usecases/`: ユーザー操作起点のシナリオ仕様（受け入れの主導線）
* `quality/`: テスト戦略、運用設定、可観測性

## Granularity Rules

* 1ファイル1責務にする（契約と挙動を同じファイルに混在させない）
* 実装詳細より先にMUSTルールを記述する
* Mermaidは改行を `\\n` ではなく `<br>` で表現する
* 各ファイルは冒頭に「目的」「スコープ」「参照」を置く

## Recommended Sections

以下の順を推奨する。

1. `目的`
2. `スコープ / 非スコープ`
3. `前提 / 参照`
4. `仕様本体`
5. `MUST / 制約`
6. `エラー / 例外`
7. `完了条件（DoD）`
8. `未決定事項`

## Naming

* ファイル名はケバブケースに統一する（例: `frontend-stream-integration.md`）
* 一時的な番号プレフィックス（`1-`, `2-` など）は使わない
* 実装言語依存の名前を避け、ドメイン責務で命名する

## Cross-Reference Rules

* 参照は原則 `act/specs/...` の絶対相対パスで書く
* `backend/` や `frontend/` から参照する場合も同じパス基準を使う
* リネーム時は `rg` で旧パスを全置換してから完了とする
