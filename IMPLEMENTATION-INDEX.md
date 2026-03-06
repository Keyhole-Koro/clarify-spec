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
3. `act/specs/overview/act-architecture.md`
4. `act/specs/context/README.md`
5. `act/specs/context/bundle-schema.md`
6. `act/specs/context/core.md`
7. `act/specs/context/implementation.md`
8. `act/specs/contracts/rpc-connect-schema.md`
9. `act/specs/contracts/gemini-vertex-response-schemas.md`
10. `act/specs/behavior/act-flow.md`
11. `act/act-api/specs/adk-service-integration.md`
12. `act/act-adk-worker/specs/act-adk-runtime.md`
13. `act/act-api/specs/session-and-auth-boundary.md`
14. `act/act-api/specs/cloudrun-redis-topology.md`
15. `act/act-api/specs/connect-server.md`
16. `act/act-api/specs/access-control-middleware.md`
17. `act/act-api/specs/runact-implementation.md`
18. `act/specs/behavior/frontend-stream-integration.md`
19. `act/specs/behavior/frontend-canvas-phases.md`
20. `act/specs/usecases/README.md`
21. `act/specs/usecases/models/usecase-er-diagrams.md`
22. `act/specs/usecases/models/usecase-logical-model.md`
23. `act/specs/usecases/graph/ask-from-empty-canvas.md`
24. `act/specs/usecases/graph/ask-with-selected-context.md`
25. `act/specs/usecases/graph/run-act-from-node-action.md`
26. `act/specs/usecases/graph/thinking-stream-visible.md`
27. `act/specs/usecases/graph/deep-research-fallback.md`
28. `act/specs/usecases/workspace/create-workspace.md`
29. `act/specs/usecases/workspace/join-workspace-by-invite-url.md`
30. `act/specs/usecases/workspace/runact-without-workspace-access.md`
31. `act/specs/quality/e2e-test-strategy.md`
32. `act/specs/quality/act-e2e-test-plan.md`
33. `act/specs/quality/llm-api-test-spec.md`
34. `act/specs/quality/hackathon-assembly-checklist.md`
35. `act/specs/quality/operations-config-observability.md`
36. `act/specs/quality/frontend-stream-acceptance.md`
37. `act/specs/quality/backend-parameter-index.md`
38. `act/specs/quality/backend-spec-gaps.md`
39. `act/act-api/backend-implementation-blueprint.md`
40. `act/act-api/README.md`
41. `act/act-api/specs/README.md`
42. `act/act-adk-worker/README.md`
43. `act/act-adk-worker/specs/README.md`

## 3. Organize 仕様索引（Source of Truth）

1. `organize/specs/README.md`
2. `organize/specs/model/topic-model.md`
3. `organize/specs/pipeline/summary.md`
4. `organize/specs/pipeline/spec.md`
5. `organize/specs/pipeline/core.md`
6. `organize/specs/pipeline/agents.md`
7. `organize/specs/pipeline/ops.md`
8. `organize/agents/README.md`
9. `organize/agents/a0/specs/ingest-extract-spec-v0.3.md`
10. `organize/agents/a1/specs/a0-a1-boundary.md`
11. `firestore/schema.md`
12. `firestore/indexes.md`

## 4. 実装手順（MDの順番）

### 手順1: Context 契約固定

1. `organize/specs/model/topic-model.md`
2. `act/specs/context/bundle-schema.md`
3. `act/specs/context/core.md`
4. `act/specs/context/implementation.md`

### 手順2: データモデル固定

1. `firestore/schema.md`
2. `firestore/indexes.md`

### 手順3: Organize 配線固定

1. `organize/specs/pipeline/core.md`
2. `organize/specs/pipeline/agents.md`
3. `organize/specs/pipeline/ops.md`

### 手順4: Act 契約固定

1. `act/specs/contracts/rpc-connect-schema.md`
2. `act/specs/overview/act-architecture.md`
3. `act/specs/behavior/act-flow.md`
4. `act/act-api/specs/adk-service-integration.md`
5. `act/act-adk-worker/specs/act-adk-runtime.md`
6. `act/act-api/specs/runact-implementation.md`
7. `act/act-api/backend-implementation-blueprint.md`
8. `act/act-api/README.md`
9. `act/act-api/specs/README.md`
10. `act/act-adk-worker/README.md`
11. `act/act-adk-worker/specs/README.md`

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

* Context仕様変更は `act/specs/context` / `organize/specs/model/topic-model.md` / `act/specs/contracts/rpc-connect-schema.md` を同時更新する
* 実装メモは `act/act-api` / `act/act-adk-worker` / `act/frontend` に置く
* 仕様の正本は `act/specs` / `organize/specs` / `firestore` とする
