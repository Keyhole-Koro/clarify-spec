# Concept (Hackathon v0.3)

このプロジェクトは、情報探索（Act）と知識整理（Organize）を分離しつつ、同じ知識グラフへ接続する。

## 目的

* 会話・検索・外部ツール情報を、抽象↔具体のノード構造で扱う
* 「探索で得た知見」を「整理された知識」へ確実に反映できる
* 根拠（evidence）を辿れる状態を維持する

## ドメイン分割

* `act/`: ユーザー対話でノード候補をリアルタイム生成（Draft中心）
* `organize/`: 入力を安定処理して永続知識グラフへ反映（Pipeline中心）
* `frontend/`: 単一画面UIの表示要件と実装方針

## 不変条件（MUST）

* 構造と表示を分離する（契約 = `act/specs/contracts`、表示 = frontend）
* at-least-once前提で冪等に処理する（重複受信で壊れない）
* DraftとCommitを分離する（Actは提案、永続化はApplyPatch）
* 認証は Firebase Auth の Googleアカウント（`google.com`）に限定する
* 認可は middleware で共通化し、`token user -> workspace membership -> tree access` を毎回検証する
* workspace はユーザーが複数作成可能で、招待URLで参加できる
* 根拠参照を保持する（URL / deep link / generation / sha256）

## Act の位置づけ

* Connect RPC server-streaming で `RunAct` を実行
* `PatchOp` は `upsert` / `append_md` のみ
* thought stream（thinkthrough）を通常回答と分離して表示可能
* Vertex AI（Gemini 3 Flash / Web Grounding / Deep Research）をプロファイルで使い分け

## Organize の位置づけ

* Agent分割（A0〜A7）で入力解釈→分解→統合→索引化
* Firestoreは構造メタ、本文は参照指向で管理
* lease / 冪等 / 失敗復旧を前提に運用可能にする

## UXの主軸

* フルスクリーンキャンバスでノードをストリーミング描画
* 複数選択コンテキストで派生探索
* 右ペインで詳細Markdownとアクション展開

## 仕様の正本

* Act索引: `act/specs/README.md`
* Organize索引: `organize/README.md`, `organize/specs/pipeline-summary.md`
* Frontend正本: `frontend/frontend-spec.md`

## 実装時の導線

ルートの実装順と索引は `IMPLEMENTATION-INDEX.md` を参照する。
