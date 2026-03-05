# Buf Config (Markdown Source of Truth)

実体 `buf.yaml` / `buf.gen.yaml` は持たず、このMarkdownを設定の正本とする。

## buf.yaml

```yaml
version: v1
name: buf.build/clarify/spec

breaking:
  use:
    - FILE

lint:
  use:
    - DEFAULT
```

## buf.gen.yaml

```yaml
version: v1

plugins:
  - plugin: buf.build/bufbuild/es
    out: ../gen/ts
    opt:
      - target=ts

  - plugin: buf.build/connectrpc/es
    out: ../gen/ts
    opt:
      - target=ts

  - plugin: buf.build/protocolbuffers/go
    out: ../gen/go
    opt:
      - paths=source_relative

  - plugin: buf.build/connectrpc/go
    out: ../gen/go
    opt:
      - paths=source_relative
```
