# Frontend 仕様（SPA / Next.js / Mock Factory）

本ドキュメントはハッカソン開発中プロジェクト向けのフロント仕様を定義する。
目的は「知識を抽象↔具体の階層ノードにして、情報アクセスを速くする」こと。
フロントは **Act（Connect RPC）** と **Organize（Firestore snapshot）** を境界として分離する。
さらに、**モック駆動でデモできるように Factory を用意**し、バックエンド/FirestoreなしでもUIが成立する開発体験を提供する。

---

### アプリ形態（必須前提）

* **SPA（単一画面）**として実装する（画面遷移を前提にしない）
* **Next.js（App Router） + TypeScript** を使用する（ただし実体は単一ページSPA）

---

### 採用ライブラリ（確定）

* reactflow
* elkjs
* Tailwind CSS
* shadcn/ui（Radix）
* Zustand
* react-markdown
* remark-gfm
* rehype-sanitize
* sonner
* firebase（Auth + Firestore）
* @connectrpc/connect / @connectrpc/connect-web

#### 「使うかも」メモ（今回は依存に入れないが、拡張余地は残す）

* cmdk / react-hotkeys-hook / fuse.js / nanoid

---

## UIリソース（デザイン実装で積極利用）

アイコン・アニメーション・UI部品は以下を優先的に参照して実装する。
MVPでは「独自SVGを最小化し、既存ライブラリ資産を多用する」。

### アイコン

* Lucide Icons: https://lucide.dev/icons/
* Lucide static SVG: https://app.unpkg.com/lucide-static@0.563.0/files/icons
* Tabler Icons: https://tabler.io/icons
* Tabler 個別アイコン例: https://tabler.io/icons/icon/copy
* Tabler Icons viewer: https://tablericons.com/

### アニメーション

* LottieFiles（free animations）: https://lottiefiles.com/free-animations/link

### グラフ/UI実装参考

* React Flow examples: https://reactflow.dev/examples
* React Flow example apps: https://github.com/xyflow/react-flow-example-apps
* D3: https://d3js.org/
* D3 gallery: https://observablehq.com/@d3/gallery
* shadcn/ui components: https://ui.shadcn.com/docs/components
* Radix UI primitives: https://www.radix-ui.com/primitives
* Vercel AI SDK intro: https://ai-sdk.dev/docs/introduction
* Vercel AI SDK docs: https://vercel.com/docs/ai-sdk

### 運用ルール（MUST）

* アイコンは Lucide / Tabler を第一候補にする
* 1画面内で icon style を混在させる場合は、線幅とサイズ基準（例: 16/20/24）を統一する
* 状態表現をアイコンのみに依存しない（テキスト/色/ラベルを併記）

---

### 技術的注意（Next.js絡みの詰まり回避）

* Firestore `onSnapshot` / Firebase Auth / ReactFlow / RPC呼び出しは基本 **クライアント側で動かす**

  * これらを含むコンポーネント/フックは **Client Component** として実装し、必要箇所に `use client` を付ける
* ReactFlowはSSRと相性問題が出る可能性があるため、詰まり回避として **dynamic import（`ssr:false`）の使用を許容**

---

## 機能ユニット（境界）

* **Act**：Connect RPCで探索/相談→「ノード候補・タグ・引用リンク・結論と理由」
* **Organize**：Firestore snapshotでツリー購読 + CRUD（rename/delete/move/merge）

---

## 画面/体験（MVP）

* 左：ノードツリー（検索、折りたたみ、ピン留め）※主表示はReactFlow
* 右：選択ノード詳細（Markdown表示＋簡易編集）
* 操作：最低限「削除・リネーム」。余裕があれば「move」「merge」

---

## 重要方針

### Markdown詳細は「固定UIコンポーネント群」を増やさない

右ペインは `MarkdownPane` 1つ中心で完結。

* ノード本文は `contentMd` として保持し、`react-markdown + remark-gfm + rehype-sanitize` で描画
* 必要なら `MarkdownEditor` と `MarkdownPreview` を追加して編集/プレビューを分割してよい
* 「根拠」「関連」などは Markdown見出し・箇条書きとして表現（専用UIは作らない）
* `node://` の内部リンクは MarkdownPane 内でリンクレンダリングのみカスタムし、選択ノード変更を実現

---

# ✅ 追加要件：Factory & Mock（必須）

Firestore/RPCが未接続でもデモできるように、**Factoryでモックデータを生成**し、`services` の実装を **Real/Mockで切替**できるようにする。

### Mock切替ルール

* `NEXT_PUBLIC_USE_MOCKS=true` のとき **完全モックモード**

  * organize：Firestore snapshot の代わりに「擬似snapshot（subscribe/unsubscribe）」でツリー更新を流す
  * act：Connect RPC の代わりに「擬似レスポンス（候補ノード/引用/結論）」を返す
* `NEXT_PUBLIC_USE_MOCKS=false` のとき **本番モード**（Firebase + Connect RPC）

### Factoryの責務

* 生成するもの：

  * Tree/Node（抽象↔具体の階層、親子、表示用タイトル）
  * Node本文Markdown（「要点」「根拠」「関連」セクションを含むサンプル）
  * ReactFlow用の nodes/edges（表示レイアウト可能な形）
  * act結果（候補ノード、引用リンクっぽいもの、理由、次アクション）
* 生成ポリシー：

  * **安定ID**（seedで再現可能）を返せること（例：`seed`引数）
  * 深さ/幅/ノード数を指定できること（例：`depth`, `branching`, `count`）
  * デモ映えする内容（抽象→具体の関係が分かる、Markdownにリンク/箇条書きがある）
  * URLは捏造OKではなく「デモ用サンプル」として `props.source="mock"` を付けるなど区別する

### services の設計ルール（DI / Swap）

* `services/act` と `services/organize` は **インターフェース（port）** を定義し、実装（real/mock）を差し替える
* 例：

  * `services/act/port.ts`（`ActPort` interface）
  * `services/act/real.ts`（Connect RPC）
  * `services/act/mock.ts`（Factoryで生成）
  * `services/act/index.ts`（ENVでreal/mockを返す）
* organize も同様に `OrganizePort` を用意し、`subscribeTree()` が real は `onSnapshot`、mock は擬似購読を返す

---

## ディレクトリ構成（Next.js App Router想定）

以下の方針で **具体的なディレクトリツリー（ファイル名まで）** を生成せよ。

* `services/act/*`：Connect RPC（real）と mock を共存
* `services/organize/*`：Firestore（real）と mock を共存（snapshot差替）
* Firebase初期化/Authは `services/firebase/*`
* ReactFlowとELKは `features/graph/*`
* 左ペイン統合は `features/knowledgeTree/*`
* 右ペインMarkdownは `features/nodeMarkdown/*`
* Factory/Mockは `mocks/` or `testing/` に集約（本体featuresに散らさない）
* ZustandはUI状態のみ（データの真実は organize側）

### ベースツリー（これを具体化する）

```txt
frontend/
  public/
  src/
    app/
      layout.tsx
      globals.css
      page.tsx

    features/
      knowledgeTree/
      graph/
      nodeMarkdown/
      actExplore/
      auth/

    services/
      firebase/
      act/
      organize/

    mocks/
      factories/
      datasets/
      organize/
      act/

    components/
      layout/
      ui/
      icons/

    lib/
      env.ts
      logger.ts
      error.ts

    gen/
      rpc/
```

---

## Firebase要件（必須）

* Google/Gmailログイン（Firebase Authの GoogleAuthProvider）
* Firestore snapshot（onSnapshot）
* onAuthStateChangedで未ログイン時はログイン誘導
* モックモード時はAuth/Firestoreを必須にしない（UIを回す）

---

## 環境変数（必須ルール）

* `NEXT_PUBLIC_*` を使用
* 必須：

  * `NEXT_PUBLIC_USE_MOCKS`（true/false）
  * `NEXT_PUBLIC_RPC_BASE_URL`
  * `NEXT_PUBLIC_FIREBASE_*` 一式
* 任意：

  * `NEXT_PUBLIC_MOCK_SEED`（同じデモ状態を再現）

---

## UI初期化の必須ポイント

* `app/layout.tsx` に sonner Toaster
* `reactflow/dist/style.css` のimport位置を決める

---

## 実装責務（守ること）

* organize購読：`features/knowledgeTree/hooks/useTreeSnapshot.ts` → `services/organize/index.ts`（real/mock吸収）
* ノード操作：`features/knowledgeTree/hooks/useTreeActions.ts` → `services/organize/index.ts`
* act呼び出し：`features/actExplore/hooks/*` → `services/act/index.ts`
* ReactFlow描画：`features/graph/components/GraphCanvas.tsx`
* ELKレイアウト：`features/graph/utils/layoutElk.ts`
* Markdown表示：`features/nodeMarkdown/components/MarkdownPane.tsx`（sanitize必須）
* Factory/Mockは `src/mocks/*` 配下に閉じ込め、本体featuresに混ぜない

---

## 設計成果物

1. 上の条件を満たす **具体的なディレクトリツリー（ファイル名まで）**
2. `dependencies` / `devDependencies` 形式の **導入する依存一覧**
3. `features/` `services/` `mocks/` `components/` の主要ディレクトリの **役割を1行ずつ**

---

## 実装フェーズ資料

実装時は以下を参照する。

* `act/specs/behavior/frontend-canvas-phases.md`（Phase 1〜3 の要件と受け入れ条件）
* `act/frontend/ai-implementation-prompts.md`（AI実装指示テンプレート）
* `rpc/connect-rpc.md`（Connect RPCの生成と接続方針）
