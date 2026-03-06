# Act Usecases

ユーザー操作起点で Act の期待動作を定義する層。
契約定義は `contracts/`、内部アルゴリズムは `behavior/` を参照し、ここでは受け入れ観点を固定する。

## Structure

* `act/specs/usecases/models/`
  * `usecase-er-diagrams.md`（usecase別ER図）
  * `usecase-logical-model.md`（論理モデル）
* `act/specs/usecases/graph/`
  * `ask-from-empty-canvas.md` (`UC-ASK-EMPTY-01`)
  * `ask-with-selected-context.md` (`UC-ASK-CONTEXT-01`)
  * `run-act-from-node-action.md` (`UC-RUNACT-NODE-01`)
  * `thinking-stream-visible.md` (`UC-THINK-STREAM-01`)
  * `deep-research-fallback.md` (`UC-DEEP-FALLBACK-01`)
* `act/specs/usecases/workspace/`
  * `create-workspace.md` (`UC-WORKSPACE-CREATE-01`)
  * `join-workspace-by-invite-url.md` (`UC-WORKSPACE-INVITE-01`)
  * `runact-without-workspace-access.md` (`UC-WORKSPACE-AUTHZ-01`)

## Rules

* 1ファイル1ユースケース
* 契約の再定義禁止（`contracts/` 参照のみ）
* E2EケースIDを末尾に必ず記載
