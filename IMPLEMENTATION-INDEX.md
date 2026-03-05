# Implementation Index

このファイルは、ハッカソン当日に迷わないための「仕様索引」と「実装順」を固定する。

## 1. まず読む（全体）

1. `concept.md`
2. `act/README.md`
3. `organize/README.md`
4. `frontend/frontend-spec.md`

## 2. Act 仕様索引（Source of Truth）

1. `act/specs/README.md`
2. `act/specs/overview/act-overview.md`
3. `act/specs/contracts/rpc-connect-schema.md`
4. `act/specs/contracts/gemini-vertex-response-schemas.md`
5. `act/specs/behavior/act-flow.md`
6. `act/specs/behavior/act-langgraph-runtime.md`
7. `act/specs/behavior/connect-server.md`
8. `act/specs/behavior/runact-implementation.md`
9. `act/specs/behavior/frontend-stream-integration.md`
10. `act/specs/behavior/frontend-canvas-phases.md`
11. `act/specs/usecases/README.md`
12. `act/specs/usecases/ask-from-empty-canvas.md`
13. `act/specs/usecases/ask-with-selected-context.md`
14. `act/specs/usecases/run-act-from-node-action.md`
15. `act/specs/usecases/thinking-stream-visible.md`
16. `act/specs/usecases/deep-research-fallback.md`
17. `act/specs/quality/e2e-test-strategy.md`
18. `act/specs/quality/act-e2e-test-plan.md`
19. `act/specs/quality/llm-api-test-spec.md`
20. `act/specs/quality/operations-config-observability.md`
21. `act/specs/quality/frontend-stream-acceptance.md`

## 3. Organize 仕様索引（Source of Truth）

1. `organize/specs/pipeline-summary.md`
2. `organize/specs/pipeline-spec.md`
3. `organize/specs/pipeline-core.md`
4. `organize/specs/pipeline-agents.md`
5. `organize/specs/pipeline-ops.md`
6. `organize/agents/README.md`
7. `organize/agents/a0/specs/ingest-extract-spec-v0.3.md`
8. `organize/agents/a1/specs/a0-a1-boundary.md`
9. `organize/data/firestore-schema.md`

## 4. 実装順（MDの順番）

### Day 1: 契約固定

1. `act/specs/contracts/rpc-connect-schema.md`
2. `act/specs/contracts/gemini-vertex-response-schemas.md`
3. `organize/data/firestore-schema.md`

### Day 2: Act Backendの骨格

1. `act/specs/behavior/connect-server.md`
2. `act/specs/behavior/runact-implementation.md`
3. `act/specs/behavior/act-langgraph-runtime.md`

### Day 3: Frontend stream接続

1. `frontend/frontend-spec.md`
2. `act/specs/behavior/frontend-stream-integration.md`
3. `act/specs/behavior/frontend-canvas-phases.md`
4. `act/specs/quality/frontend-stream-acceptance.md`

### Day 4: Usecase成立確認

1. `act/specs/usecases/ask-from-empty-canvas.md`
2. `act/specs/usecases/ask-with-selected-context.md`
3. `act/specs/usecases/run-act-from-node-action.md`
4. `act/specs/usecases/thinking-stream-visible.md`
5. `act/specs/usecases/deep-research-fallback.md`

### Day 5: Organize接続

1. `organize/specs/pipeline-core.md`
2. `organize/specs/pipeline-agents.md`
3. `organize/specs/pipeline-ops.md`

### Day 6: 品質・運用仕上げ

1. `act/specs/quality/e2e-test-strategy.md`
2. `act/specs/quality/act-e2e-test-plan.md`
3. `act/specs/quality/llm-api-test-spec.md`
4. `act/specs/quality/operations-config-observability.md`

## 5. ルール

* 契約変更は `contracts` を先に更新し、他ファイルは追従
* 実装メモは `act/backend` / `act/frontend` に置く
* 仕様は `act/specs` / `organize/specs` を正本にする
