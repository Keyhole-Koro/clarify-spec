# Act Streaming Guide（Human View）

## まず覚えること

* `RunAct` は server-streaming
* イベントは `RunActEvent`
* `done` と `error` は排他
* `patch_ops` は `upsert` / `append_md` のみ
* 本文 state の正本は `append_md`
* サーバ契約では `done` と `error` は厳密排他

## 主要ケース

1. 通常回答
* `upsert` -> `append_md`... -> `done`

2. thinkingあり
* `stream_parts(thought=true)` -> `stream_parts(thought=false)` -> `patch_ops` -> `done`

3. 途中エラー
* 部分結果が表示済みでも `error` で終端（巻き戻しなし）

4. 同一 `request_id` 再送
* 二重実行防止（`ALREADY_EXISTS` またはキャッシュ結果）

5. クライアント切断
* `act-api` が worker context を cancel し、以降の event は増えない

## 順序保証

バグを減らすため、stream には順序の前提を置く。

* 初回の本文系 `append_md` より先に対象 block の `upsert` が来る
* 同一 event 内の基本順序は `thought -> answer -> upsert -> append_md -> metadata -> terminal`
* `done/error` は常に最後
* metadata は本文生成を止めない

## terminal の扱い

契約としては:

* 正常系は最後に `done` だけ送る
* 異常系は最後に `error` だけ送る
* 終端後に追加 event は送らない

frontend は防御的に:

* 最初の terminal だけ採用する
* 後から来た duplicate terminal は無視する
* UI は最初の終端結果だけを反映する
* dev console には warning を残せるようにする

## thinkthrough の扱い

thought は answer 本文とは別物として扱う。

* thought 表示の既定は OFF
* `thinking_config.include_thoughts=true` の request でのみ表示対象になる
* thought は request 単位の別 buffer に保持する
* node 本文には混ぜない
* 実行中は展開表示できる
* `done/error` 後は既定で折りたたむ
* 再実行時に前回 request の thought を引き継がない

この前提があるので、frontend は:

* thought buffer
* answer buffer
* node state (`append_md`)

を分けて持てばよい。

## cancel 伝播

切断時は「途中まで出たものは有効、以降は止める」と考える。

* client disconnect または deadline で `act-api` が worker context を cancel する
* worker は model / grounding / tool / polling まで同じ cancel を伝播する
* cancel 後は新しい event を生成しない
* 既送信イベントは巻き戻さない
* reconnect は resume ではなく再実行

## Deep Research fallback の見え方

Deep Research が timeout や一時失敗で継続不能になっても、UI では「失敗」より「継続中の切替」として扱う。

* fallback はエラー通知ではなく、軽いインライン状態表示にする
* ユーザーには「標準モードへ切り替えて継続中」と伝える
* stream 自体は同一 request のまま継続する
* fallback reason の詳細は dev console / log に残す

## model profile の基本ルール

まずは単純に分ける。

* 既定は `Flash`
* `research_config.use_deep_research=true` のときだけ `Deep Research`
* grounding は profile と独立に ON/OFF する
* `Deep Research` が継続不能なら同一 request 内で `Flash` へ fallback する

つまり:

* profile の既定 = `Flash`
* 調査モードの明示指定 = `Deep Research`
* grounding = 追加能力

## metadata の見え方

metadata は本文の主表示とは分ける。

* grounding metadata は本文下の `References` に出す
* tool metadata は既定で `Diagnostics` に寄せる
* 本文生成を block せず、本文レイアウトに混ぜない
* 未対応項目も store と dev console では追跡できるようにする
* SDK の raw payload 全量は UI 契約へそのまま出さない

## フロント実装の最低ルール

* thought と answer は別state
* `append_md` は追記のみ
* `append_md` を node 本文 state の正本として扱う
* error時は `stage/retryable` で再試行UIを分岐
* duplicate terminal が来ても UI を反転させない

## 正本参照

* `act/specs/behavior/ai-streaming-cases.md`
* `act/specs/behavior/frontend-stream-integration.md`
* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runtime/runact-implementation.md`
* `act/act-adk-worker/specs/act-adk-runtime.md`
