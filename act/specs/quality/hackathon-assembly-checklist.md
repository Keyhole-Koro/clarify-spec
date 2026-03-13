# Assembly Checklist

## 目的

ローカルおよび本番前確認で、実装者が順番どおりに組み立てて最短で動作確認できるようにする。

## スコープ / 非スコープ

* スコープ: Act領域の当日実装・確認手順
* 非スコープ: 長期運用チューニング

## 前提 / 参照

* `act/act-api/specs/security/session-and-auth-boundary.md`
* `act/act-api/specs/platform/cloudrun-redis-topology.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/quality/backend-parameter-index.md`

## 手順（当日実装順）

1. Firebase Auth + Google provider を有効化
2. Cloud Run service を起動（Redis到達性を前提）
3. Memorystore接続を有効化
4. sid Cookie発行と csrf token発行を確認
5. `RunAct` で `request_id` 付き送信を確認
6. `ErrorInfo.stage/retryable` をUI表示へ反映
7. Redis停止時の fail-closed と復旧動作を確認

## 動作確認項目（最小E2E）

* token有効 + sid有効 + csrf一致: 成功
* token有効 + sid欠落: `UNAUTHENTICATED`
* csrf不一致: 拒否
* workspace未所属: `PERMISSION_DENIED`
* tree/workspace不整合: `PERMISSION_DENIED`
* Redis不達: `UNAVAILABLE` + stage/retryable付与

## 失敗時切り分け（Auth / Redis / Firestore / LLM）

1. `stage=AUTHN` -> token/provider確認
2. `stage=SID_VALIDATE` -> Redis疎通・sid TTL確認
3. `stage=AUTHZ` -> membership/tree所属確認
4. `stage=LOAD_CONTEXT` -> Firestore読み取り確認
5. `stage=GENERATE_WITH_MODEL` -> Vertex呼び出し確認

## 数値パラメータ

* `SID_TTL_SECONDS=86400`
* `SID_REQ_TTL_SECONDS=900`
* `SID_LOCK_TTL_SECONDS=10`
* `CSRF_TTL_SECONDS=86400`

## 受け入れ条件（DoD）

* 最小E2Eケースを当日再現できる
* 障害時に stage ベースで切り分けできる
* Redis 復旧後の再ログイン / 再試行フローが確認できる

## 実装メモ（最小）

* Redis 障害時は復旧完了まで RunAct を止める
* 復旧後は sid 再発行と再試行導線を確認する
