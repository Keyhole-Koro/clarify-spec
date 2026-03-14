# Run Phase Contract

Version: 1.0

目的: `run-phase` CLI の共通 I/O と mode を固定する。

## 1. CLI Surface

基本形:

```bash
run-phase <phase> --input <path> --mode <mode>
```

必須引数:

* `phase`
* `--input`

任意引数:

* `--mode`
* `--expected`
* `--llm-mode`
* `--output`
* `--update-golden`

## 2. `phase`

許可値:

* `a0`
* `a1`
* `topic-resolver`
* `a2`
* `a3b`
* `a6`
* `a3`
* `a4`
* `a7`
* `a5`

## 3. `mode`

許可値:

* `preview`
* `apply-local`
* `golden-check`

意味:

* `preview`
  * 副作用を実行せず preview object を返す
* `apply-local`
  * emulator に対して write する
* `golden-check`
  * `expected.json` と比較し diff を返す

## 4. `llm-mode`

許可値:

* `stub-llm`
* `real-llm`

default:

* `stub-llm`

## 5. Input Contract

入力 file は phase ごとに異なるが、共通 envelope を持ってよい。

```json
{
  "meta": {
    "workspaceId": "ws_demo",
    "topicId": "tp_ai_agents",
    "traceId": "tr_local_001"
  },
  "input": {}
}
```

`input` の shape は phase ごとの fixture 定義に従う。

## 6. Output Contract

`run-phase` の共通出力 shape:

```json
{
  "phase": "a1",
  "mode": "preview",
  "llmMode": "stub-llm",
  "result": {},
  "events": [],
  "firestoreWrites": [],
  "gcsWrites": [],
  "diagnostics": [],
  "diff": null
}
```

## 7. Preview Objects

### 7.1 Firestore Write

```json
{
  "path": "workspaces/ws_demo/topics/tp_ai_agents/nodes/nd_001",
  "operation": "upsert",
  "data": {}
}
```

### 7.2 GCS Write

```json
{
  "path": "mind/node_rollup/nd_001/v1.html",
  "contentType": "text/html",
  "bodyPreview": "<h1>...</h1>",
  "sha256": "..."
}
```

### 7.3 Event Preview

```json
{
  "type": "draft.updated",
  "workspaceId": "ws_demo",
  "topicId": "tp_ai_agents",
  "payload": {}
}
```

## 8. Diff Contract

`golden-check` のときだけ `diff` を返してよい。

```json
{
  "status": "match | mismatch",
  "sections": ["result", "events", "firestoreWrites", "gcsWrites"],
  "summary": "..."
}
```

## 9. Exit Code

* `0`
  * preview / apply-local 成功
  * golden-check で一致
* `1`
  * 実行失敗
* `2`
  * golden mismatch

## 10. MUST

* `preview` は副作用を発生させない
* `golden-check` は `expected.json` が無ければ失敗する
* `stub-llm` では外部 API を呼ばない
* `apply-local` でも preview object を返す
