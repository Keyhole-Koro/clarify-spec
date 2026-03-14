# Organize Precision Improvement Tickets

Version: 1.0

目的: 精度不安のある Agent を、優先度付きの改善チケットとして管理する。

進め方:

* 1 ticket ずつ仕様化してから実装へ進める
* 先に upstream の精度を上げ、後段の評価はそのあとに行う
* 完了した ticket は本ファイルから削除し、正本 spec へ反映する

## Ticket OGT-001: A1 Atom 粒度の安定化

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a1/README.md)

問題:

* 1 文に複数 claim がある場合に atom 分割がぶれる
* `kind` と `confidence` の付与が毎回揺れる
* 後段の A3 Cleaner が atom 品質の揺れをそのまま引き受ける

状態:

* resolved

反映先:

* [a1-atom-stability.md](/home/unix/clarify-spec/organize/specs/pipeline/a1-atom-stability.md)
* [README.md](/home/unix/clarify-spec/organize/agents/a1/README.md)
* [agents.md](/home/unix/clarify-spec/organize/specs/pipeline/agents.md)

## Ticket OGT-002: A7 contextSummary の安定化

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a7/README.md)

問題:

* Act の Context Assembly に入る `contextSummary` が短すぎて情報落ちしやすい
* detailHtml と contextSummary の役割境界が弱い
* 重要ノードなのに毎回違う観点で要約される可能性がある

やること:

* `contextSummary` の出力テンプレートを固定する
* 必須要素を `what / why / relation` の 3 要素に分ける
* title, kind, parent, top evidence を deterministic に前処理で抽出する
* Gemini は圧縮だけを担当する
* Act 注入用 summary と UI 用 detailHtml の評価軸を分離する

受け入れ条件:

* 同じ node で summary の主語や焦点が大きくぶれない
* Act 注入に必要な relation 情報が欠落しない
* detailHtml の揺れがあっても contextSummary は安定する

優先度:

* P0

## Ticket OGT-003: A0 抽出品質の安定化

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a0/README.md)

問題:

* PDF / 画像 / HTML の抽出品質が入力形式に強く依存する
* OCR や表抽出のぶれが A1 にそのまま流れる
* 「抽出失敗」なのか「原文にそう書いてある」のかが曖昧になる

やること:

* sourceType ごとの quality flag を出す
* OCR 系は confidence を保持する
* 表・見出し・箇条書きの抽出をテンプレート化する
* A1 へ渡す前に構造崩れを検知する validation を追加する
* heavy extract の golden input を用意する

受け入れ条件:

* HTML/PDF/画像の代表入力で構造崩れが検知できる
* 抽出品質が低い入力を downstream に無警告で流さない
* evidence trace から抽出元を追える

優先度:

* P1

## Ticket OGT-004: A4 keyword / ranking 再現性の安定化

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a4/README.md)

問題:

* keyword 抽出を LLM に寄せすぎると index の再現性が落ちる
* relevance 以外の feature と keyword の責務が混ざりやすい
* retrieval 品質の揺れが topic resolution や Act ranking に伝播する

やること:

* keyword 抽出を deterministic 優先に寄せる
* title/body/ngram の機械抽出を primary にする
* Gemini は補助タグ生成だけに限定する
* ranking feature の算出入力を固定する
* 同一 outlineVersion に対する index 差分を比較できる fixture を持つ

受け入れ条件:

* 同一入力に対して keyword の主要集合が安定する
* ranking feature の再計算で大きなジャンプが起きにくい
* LLM 不調時でも最低限の index が維持できる

優先度:

* P1

## Ticket OGT-005: TopicResolver 判定の誤 attach 抑制

対象:

* [topic-resolution.md](/home/unix/clarify-spec/organize/specs/pipeline/topic-resolution.md)

問題:

* 完全自動化すると近い topic への誤 attach が起きうる
* 候補 retrieval と Gemini 判定の責務がまだ粗い

やること:

* candidate retrieval の deterministic score を明文化する
* Gemini 判定の structured output schema を固定する
* `attach_existing` の最低 confidence を決める
* borderline case 用 fixture を作る

受け入れ条件:

* 近接 topic 群で誤 attach より `create_new` が優先される
* workspace 外 topic は絶対に候補に入らない
* 判定理由を trace で説明できる

優先度:

* P1

## Ticket OGT-006: A3 Merge 監査モデルの明確化

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a3/README.md)

問題:

* `MERGE / CREATE / REJECT / DEFER` はあるが、merge 判断の監査モデルが弱い
* aliases, merge reason, supersede 履歴の保存先が曖昧
* 誤 merge を後から追跡・修正しにくい

やること:

* merge decision の永続 shape を定義する
* `aliases`, `merged_from`, `superseded_claims`, `merge_reason` の保存先を決める
* node 単位の merge audit trail を持たせる
* A3 preview で merge diff と merge rationale を見られるようにする

受け入れ条件:

* 任意の node merge について、なぜ merge されたかを trace できる
* alias の出所が分かる
* 誤 merge の分析に必要な情報が Firestore / log / preview のいずれかに残る

優先度:

* P0

## Ticket OGT-007: Topic Lifecycle の定義

対象:

* [topic-resolution.md](/home/unix/clarify-spec/organize/specs/pipeline/topic-resolution.md)

問題:

* `attach_existing | create_new` の先が未定義
* topic が増えすぎた場合の `merge / split / archive / rename` がない
* 完全自動 routing だと長期運用で topic 境界が崩れやすい

やること:

* topic lifecycle state を定義する
* `active / archived / merged / split` 等の状態を決める
* topic merge / split の trigger と運用責務を決める
* TopicResolver が archived topic をどう扱うかを固定する

受け入れ条件:

* 増えすぎた topic を後から整理できる
* archived / merged topic が routing 候補にどう入るか説明できる
* topic lifecycle が access boundary と矛盾しない

優先度:

* P0

## Ticket OGT-008: `organizeOps` の消費者定義

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a5/README.md)

問題:

* `organizeOps/{opId}` は生成されるが、誰が消費するかが未定
* 提案で止めるのか、自動適用するのかが不明
* UI 露出や承認フローがない

やること:

* `organizeOps` の state machine を定義する
* `planned / approved / applied / dismissed` などを決める
* 消費者を `human review`, `auto-applier`, `frontend` のどれにするか固定する
* `topic.node_changed` などとの連動条件を決める

受け入れ条件:

* `organizeOps` が単なる墓場にならない
* 各 op が誰によりどう消費されるか説明できる
* 自動適用の有無が phase ごとに明確である

優先度:

* P1

## Ticket OGT-009: Schema Migration Policy

対象:

* [README.md](/home/unix/clarify-spec/organize/agents/a3b/README.md)
* [README.md](/home/unix/clarify-spec/organize/agents/a3/README.md)

問題:

* `schema_version` は進化できるが、既存 node / edge / index の再計算方針が薄い
* 旧 schema 産物と新 schema 産物の共存期間が未定
* migration の実行主体がない

やること:

* schema migration policy を定義する
* `lazy migration` か `eager rebuild` かを決める
* index / rollup / outline の再生成トリガーを決める
* 旧 schema 参照データの互換期間を決める

受け入れ条件:

* schema_version が進んでも既存 topic が壊れない
* node / edge / index がどの schema_version に属するか辿れる
* migration の実行責務が明確である

優先度:

* P1

## Ticket OGT-010: Phase 別 Fixture 実例の固定

対象:

* [run-phase-contract.md](/home/unix/clarify-spec/organize/specs/pipeline/run-phase-contract.md)
* [minimal-fixture-set.md](/home/unix/clarify-spec/organize/specs/pipeline/minimal-fixture-set.md)

問題:

* contract と最小セットはあるが、phase ごとの実例 fixture shape がまだない
* 実装者が `input.json` と `expected.json` をどう書くかで迷う

やること:

* `a1`, `topic-resolution`, `a3b`, `a3` の fixture 実例を 1 ケースずつ固定する
* `input.json` / `expected.json` のサンプル shape を文書化する
* `golden-check` の想定 diff を例示する

受け入れ条件:

* phase ごとに 1 つはそのまま模写できる fixture 例がある
* `run-phase` 実装者が schema を推測しなくて済む
* fixture のレビュー単位が揃う

優先度:

* P1
