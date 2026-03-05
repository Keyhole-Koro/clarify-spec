# Act Usecases

ユーザー操作起点で Act の期待動作を定義する層。
契約定義は `contracts/`、内部アルゴリズムは `behavior/` を参照し、ここでは受け入れ観点を固定する。

## Usecase List

* `act/specs/usecases/usecase-er-diagrams.md`（usecase別ER図）
* `act/specs/usecases/usecase-logical-model.md`（論理モデル）
* `act/specs/usecases/ask-from-empty-canvas.md` (`UC-ASK-EMPTY-01`)
* `act/specs/usecases/ask-with-selected-context.md` (`UC-ASK-CONTEXT-01`)
* `act/specs/usecases/run-act-from-node-action.md` (`UC-RUNACT-NODE-01`)
* `act/specs/usecases/thinking-stream-visible.md` (`UC-THINK-STREAM-01`)
* `act/specs/usecases/deep-research-fallback.md` (`UC-DEEP-FALLBACK-01`)
* `act/specs/usecases/create-workspace.md` (`UC-WORKSPACE-CREATE-01`)
* `act/specs/usecases/join-workspace-by-invite-url.md` (`UC-WORKSPACE-INVITE-01`)
* `act/specs/usecases/runact-without-workspace-access.md` (`UC-WORKSPACE-AUTHZ-01`)

## Rules

* 1ファイル1ユースケース
* 契約の再定義禁止（`contracts/` 参照のみ）
* E2EケースIDを末尾に必ず記載
