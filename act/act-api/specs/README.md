# Act API Specs

Go `act-api` 実装に依存する仕様を置く。

## Categories

### `runtime/`

* `runtime/connect-server.md`
* `runtime/runact-implementation.md`
* `runtime/adk-service-integration.md`

### `security/`

* `security/session-and-auth-boundary.md`
* `security/access-control-middleware.md`
* `security/internal-service-auth.md`
* `security/cookie-cors-policy.md`

### `platform/`

* `platform/cloudrun-redis-topology.md`
* `platform/redis-keyspace-policy.md`

### `reliability/`

* `reliability/idempotency-policy.md`
* `reliability/error-mapping-matrix.md`

### `workspace/`

* `workspace/invite-link-api-contract.md`

## Rule

* 契約正本は `act/specs/contracts/*` を参照する
* Context正本は `act/specs/context/*` を参照する
* 共有フローは `act/specs/behavior/act-flow.md` を参照する
