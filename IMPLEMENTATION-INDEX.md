# Implementation Index

このファイルは、ハッカソン当日に迷わないための「仕様索引」と「実装順」を固定する。

## 1. まず読む（全体）

1. `concept.md`
2. `specs/shared/README.md`
3. `act/README.md`
4. `organize/README.md`
5. `frontend/frontend-spec.md`

## 2. Shared 仕様索引（Source of Truth）

1. `specs/shared/README.md`
2. `specs/shared/topic-model.md`
3. `specs/shared/context-bundle-schema.md`
4. `specs/shared/context-assembly-core.md`

## 3. Act 仕様索引（Source of Truth）

1. `act/specs/README.md`
2. `act/specs/overview/act-overview.md`
3. `act/specs/overview/act-architecture.md`
4. `act/specs/contracts/rpc-connect-schema.md`
5. `act/specs/contracts/gemini-vertex-response-schemas.md`
6. `act/specs/behavior/act-flow.md`
7. `act/specs/behavior/act-langgraph-runtime.md`
8. `act/specs/behavior/session-and-auth-boundary.md`
9. `act/specs/behavior/cloudrun-redis-topology.md`
10. `act/specs/behavior/connect-server.md`
11. `act/specs/behavior/access-control-middleware.md`
12. `act/specs/behavior/runact-implementation.md`
13. `act/specs/behavior/frontend-stream-integration.md`
14. `act/specs/behavior/frontend-canvas-phases.md`
15. `act/specs/usecases/README.md`
16. `act/specs/usecases/usecase-er-diagrams.md`
17. `act/specs/usecases/usecase-logical-model.md`
18. `act/specs/usecases/ask-from-empty-canvas.md`
19. `act/specs/usecases/ask-with-selected-context.md`
20. `act/specs/usecases/run-act-from-node-action.md`
21. `act/specs/usecases/thinking-stream-visible.md`
22. `act/specs/usecases/deep-research-fallback.md`
23. `act/specs/usecases/create-workspace.md`
24. `act/specs/usecases/join-workspace-by-invite-url.md`
25. `act/specs/usecases/runact-without-workspace-access.md`
26. `act/specs/quality/e2e-test-strategy.md`
27. `act/specs/quality/act-e2e-test-plan.md`
28. `act/specs/quality/llm-api-test-spec.md`
29. `act/specs/quality/hackathon-assembly-checklist.md`
30. `act/specs/quality/operations-config-observability.md`
31. `act/specs/quality/frontend-stream-acceptance.md`
32. `act/specs/quality/backend-parameter-index.md`
33. `act/specs/quality/backend-spec-gaps.md`

## 4. Organize 仕様索引（Source of Truth）

1. `organize/specs/pipeline-summary.md`
2. `organize/specs/pipeline-spec.md`
3. `organize/specs/pipeline-core.md`
4. `organize/specs/pipeline-agents.md`
5. `organize/specs/pipeline-ops.md`
6. `organize/agents/README.md`
7. `organize/agents/a0/specs/ingest-extract-spec-v0.3.md`
8. `organize/agents/a1/specs/a0-a1-boundary.md`
9. `firestore/schema.md`
10. `firestore/indexes.md`

## 5. 実装手順（MDの順番）

### 手順1: Shared 契約固定

1. `specs/shared/topic-model.md`
2. `specs/shared/context-bundle-schema.md`
3. `specs/shared/context-assembly-core.md`

### 手順2: データモデル固定

1. `firestore/schema.md`
2. `firestore/indexes.md`

### 手順3: Organize 配線固定

1. `organize/specs/pipeline-core.md`
2. `organize/specs/pipeline-agents.md`
3. `organize/specs/pipeline-ops.md`

### 手順4: Act 契約固定

1. `act/specs/contracts/rpc-connect-schema.md`
2. `act/specs/overview/act-architecture.md`
3. `act/specs/behavior/act-flow.md`
4. `act/specs/behavior/runact-implementation.md`

### 手順5: Frontend Stream接続

1. `frontend/frontend-spec.md`
2. `frontend/frontend-ui-gaps.md`
3. `act/specs/behavior/frontend-stream-integration.md`
4. `act/specs/behavior/frontend-canvas-phases.md`
5. `act/specs/quality/frontend-stream-acceptance.md`

### 手順6: 品質・運用仕上げ

1. `act/specs/quality/e2e-test-strategy.md`
2. `act/specs/quality/act-e2e-test-plan.md`
3. `act/specs/quality/llm-api-test-spec.md`
4. `act/specs/quality/hackathon-assembly-checklist.md`
5. `act/specs/quality/operations-config-observability.md`
6. `act/specs/quality/backend-parameter-index.md`
7. `act/specs/quality/backend-spec-gaps.md`

## 6. ルール

* 共有仕様変更は `specs/shared` を先に更新し、Act/Organizeを追従させる
* 実装メモは `act/backend` / `act/frontend` に置く
* 仕様の正本は `specs/shared` / `act/specs` / `organize/specs` / `firestore` とする
