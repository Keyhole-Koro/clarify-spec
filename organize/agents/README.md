# Organize Agent Specs

Agentごとの詳細仕様。  
各 Agent はディレクトリ単位で管理し、入口は `README.md` に統一する。

## Structure

* `a0/`
  * `README.md`: A0 の責務、I/O、競合対策
  * `specs/`: ingest/extract の詳細仕様
* `a1/`
  * `README.md`: A1 の責務、I/O、競合対策
  * `specs/`: A0-A1 境界契約
* `a2/`
  * `README.md`: A2 の責務、I/O、競合対策
* `a3b/`
  * `README.md`: A3b の責務、I/O、schema proposal
* `a3/`
  * `README.md`: A3 の責務、処理段階、競合対策
* `a4/`
  * `README.md`: A4 の責務、I/O、競合対策
* `a5/`
  * `README.md`: A5 の責務、I/O、競合対策
* `a6/`
  * `README.md`: A6 の責務、I/O、競合対策
* `a7/`
  * `README.md`: A7 の責務、I/O、競合対策

## Reading Order

1. `organize/specs/pipeline/agents.md`
2. `organize/agents/a3/README.md`
3. `organize/agents/a3b/README.md`
4. `organize/agents/a4/README.md`
5. `organize/agents/a7/README.md`
6. 必要に応じて各 `specs/`

## Agent Index

* `organize/agents/a0/README.md`
* `organize/agents/a1/README.md`
* `organize/agents/a2/README.md`
* `organize/agents/a3b/README.md`
* `organize/agents/a3/README.md`
* `organize/agents/a4/README.md`
* `organize/agents/a5/README.md`
* `organize/agents/a6/README.md`
* `organize/agents/a7/README.md`

## Quick Map

| Agent | Input | Main Output | Main Emit | Focus |
| --- | --- | --- | --- | --- |
| A0 | `media.received` | `inputs/{inputId}`, `mind/inputs/...` | `input.received` | ingest / extract |
| A1 | `input.received` | `atoms/{atomId}`, `mind/atoms/...` | `atom.created` | atomize |
| A2 | `atom.created` | draft version, `mind/drafts/...` | `draft.updated` | route to draft |
| A3b | `draft.updated` | `pipelineBundles/{bundleId}`, optional schema update | `bundle.created`, optional `topic.schema_updated` | distill / schema proposal |
| A3 | `bundle.created` | outline, nodes, edges | `outline.updated`, `topic.node_changed`, `atom.reissued` | clean / merge / commit |
| A4 | `outline.updated` | `index_items/*`, `mind/maps/...` | optional | index / map |
| A5 | `topic.metrics.updated` | `organizeOps/{opId}` | `topic.node_changed`, `topic.metrics.updated` | rebalance |
| A6 | `bundle.created` | `mind/bundle_desc/...`, `bundles/{bundleId}.descRef` | optional `bundle.described` | explain bundle |
| A7 | `topic.node_changed` | `mind/node_rollup/...`, `rollupRef` | optional `node.rollup.updated` | node summary |
