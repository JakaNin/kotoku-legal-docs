# アーキテクチャ設計書

## 🏗️ システムアーキテクチャ概要

```
┌─────────────────────────────────────────────────────────┐
│                     User Devices                        │
│              (Mobile / Desktop Browsers)                │
└────────────────────┬────────────────────────────────────┘
                     │ HTTPS
┌────────────────────▼────────────────────────────────────┐
│                  Vercel (CDN + Edge)                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │          Next.js 14 App Router                   │  │
│  │  - Server Components (RSC)                       │  │
│  │  - Server Actions                                 │  │
│  │  - API Routes (minimal)                          │  │
│  └──────────────────────────────────────────────────┘  │
└────────────┬─────────────────────────┬──────────────────┘
             │                         │
             │                         │ Webhooks
             │                         │
┌────────────▼──────────┐    ┌────────▼─────────────────┐
│   Supabase (BaaS)     │    │   Stripe                 │
│  ┌──────────────────┐ │    │  - PaymentLink           │
│  │ PostgreSQL       │ │    │  - Webhook Events        │
│  │ (with RLS)       │ │    └──────────────────────────┘
│  └──────────────────┘ │
│  ┌──────────────────┐ │
│  │ Auth (OAuth)     │ │
│  └──────────────────┘ │
│  ┌──────────────────┐ │
│  │ Storage (S3)     │ │
│  └──────────────────┘ │
│  ┌──────────────────┐ │
│  │ Realtime         │ │
│  └──────────────────┘ │
│  ┌──────────────────┐ │
│  │ Edge Functions   │ │
│  └──────────────────┘ │
└───────────────────────┘
```

---

## 🧰 技術スタック

### フロントエンド

| 技術                | バージョン | 用途                                     |
| ------------------- | ---------- | ---------------------------------------- |
| **Next.js**         | 14.x       | フルスタックフレームワーク（App Router） |
| **React**           | 18.x       | UIライブラリ                             |
| **TypeScript**      | 5.x        | 型安全性確保                             |
| **Tailwind CSS**    | 3.x        | スタイリング                             |
| **shadcn/ui**       | latest     | UIコンポーネントライブラリ               |
| **React Hook Form** | 7.x        | フォーム管理                             |
| **Zod**             | 3.x        | スキーマバリデーション                   |
| **SWR**             | 2.x        | データフェッチング・キャッシング         |

### バックエンド

| 技術                  | バージョン | 用途                                |
| --------------------- | ---------- | ----------------------------------- |
| **Supabase**          | latest     | BaaS（DB, Auth, Storage, Realtime） |
| **PostgreSQL**        | 15.x       | データベース（Supabase管理）        |
| **Supabase Auth**     | -          | 認証（Google OAuth）                |
| **Supabase Storage**  | -          | 画像ストレージ                      |
| **Supabase Realtime** | -          | WebSocketベースリアルタイム通信     |

### 決済・通知

| 技術       | 用途                       |
| ---------- | -------------------------- |
| **Stripe** | 決済処理（PaymentLink）    |
| **Resend** | トランザクションメール送信 |

### デプロイ・インフラ

| 技術               | 用途                           |
| ------------------ | ------------------------------ |
| **Vercel**         | Next.jsホスティング            |
| **GitHub Actions** | CI/CD（型チェック・lint）      |
| **Sentry**         | エラートラッキング（将来導入） |

---

## 📁 フォルダ構成

```
okaimono/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # 認証グループ
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── signup/
│   │       └── page.tsx
│   ├── (main)/                   # メインアプリグループ
│   │   ├── layout.tsx            # 共通レイアウト（ナビゲーション）
│   │   ├── page.tsx              # タイムライン（トップページ）
│   │   ├── requests/             # 依頼関連
│   │   │   ├── [id]/
│   │   │   │   ├── page.tsx      # 依頼詳細
│   │   │   │   └── chat/
│   │   │   │       └── page.tsx  # チャット
│   │   │   └── new/
│   │   │       └── page.tsx      # 新規投稿
│   │   ├── handover/             # 受け渡し
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   │   ├── reviews/              # レビュー
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   │   └── profile/              # マイページ
│   │       ├── page.tsx
│   │       └── edit/
│   │           └── page.tsx
│   ├── api/                      # API Routes（最小限）
│   │   ├── webhooks/
│   │   │   └── stripe/
│   │   │       └── route.ts
│   │   └── health/
│   │       └── route.ts
│   ├── layout.tsx                # ルートレイアウト
│   └── globals.css
│
├── components/                   # Reactコンポーネント
│   ├── features/                 # フィーチャー別コンポーネント
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx
│   │   │   └── ProfileSetup.tsx
│   │   ├── requests/
│   │   │   ├── RequestCard.tsx
│   │   │   ├── RequestForm.tsx
│   │   │   ├── RequestDetail.tsx
│   │   │   └── RequestTimeline.tsx
│   │   ├── chat/
│   │   │   ├── ChatMessage.tsx
│   │   │   ├── ChatInput.tsx
│   │   │   └── ImageUpload.tsx
│   │   ├── handover/
│   │   │   ├── ReceiptUpload.tsx
│   │   │   └── ConfirmationChecklist.tsx
│   │   └── reviews/
│   │       ├── StarRating.tsx
│   │       └── ReviewForm.tsx
│   ├── ui/                       # shadcn/ui基本コンポーネント
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── input.tsx
│   │   ├── textarea.tsx
│   │   ├── dialog.tsx
│   │   ├── toast.tsx
│   │   └── ...
│   └── layout/                   # レイアウトコンポーネント
│       ├── Header.tsx
│       ├── Navigation.tsx
│       └── Footer.tsx
│
├── lib/                          # ユーティリティ・設定
│   ├── supabase/
│   │   ├── client.ts             # Supabaseクライアント
│   │   ├── server.ts             # サーバー側クライアント
│   │   ├── middleware.ts         # ミドルウェア
│   │   └── types.ts              # DB型定義（自動生成）
│   ├── stripe/
│   │   ├── client.ts
│   │   └── webhooks.ts
│   ├── validations/              # Zodスキーマ
│   │   ├── request.ts
│   │   ├── user.ts
│   │   └── review.ts
│   ├── hooks/                    # カスタムフック
│   │   ├── useAuth.ts
│   │   ├── useRequests.ts
│   │   ├── useChat.ts
│   │   └── useRealtime.ts
│   └── utils/
│       ├── constants.ts          # 定数（禁止品目リスト等）
│       ├── format.ts             # フォーマット関数
│       └── cn.ts                 # classname utility
│
├── supabase/                     # Supabase設定
│   ├── migrations/               # DBマイグレーション
│   │   └── 20250101000000_initial_schema.sql
│   ├── functions/                # Edge Functions
│   │   └── validate-prohibited-items/
│   │       └── index.ts
│   └── config.toml
│
├── public/                       # 静的ファイル
│   ├── images/
│   └── icons/
│
├── docs/                         # プロジェクトドキュメント
│   ├── mvp-features.md
│   ├── architecture.md
│   ├── data-model.md
│   └── implementation-roadmap.md
│
├── .env.local                    # 環境変数（ローカル）
├── .env.example                  # 環境変数テンプレート
├── next.config.js
├── tsconfig.json
├── tailwind.config.js
├── package.json
└── README.md
```

---

## 🔐 認証・認可設計

### 認証フロー

1. **Google OAuth（Supabase Auth）**
   - ユーザーはGoogleアカウントでログイン
   - 初回ログイン時にプロフィール登録（名前、エリア、写真）
   - セッション管理はSupabaseが自動処理

2. **セッション管理**
   - Supabase AuthのJWT（`access_token`, `refresh_token`）
   - Next.js Middlewareで保護ルート制御
   - クッキーベースのセッション（HttpOnly）

### 認可設計（Row Level Security）

Supabase PostgreSQLのRLS（Row Level Security）でアクセス制御：

```sql
-- 例: requestsテーブルのRLSポリシー
-- 自分の投稿または自分のエリアの投稿のみ閲覧可能
CREATE POLICY "Users can view own area requests"
ON requests FOR SELECT
USING (
  auth.uid() = user_id
  OR address_area = (SELECT address_area FROM users WHERE id = auth.uid())
);

-- 自分の投稿のみ更新可能
CREATE POLICY "Users can update own requests"
ON requests FOR UPDATE
USING (auth.uid() = user_id);
```

---

## 💾 データフェッチング戦略

### Server Components（デフォルト）

- 初回ページロードはServer Componentsで静的データ取得
- SEO対応不要なページも高速レンダリング

### Client Components + SWR

- リアルタイム性が必要な箇所（タイムライン、チャット）
- SWRでキャッシング＋自動再検証

### Server Actions

**用途:**

- フォーム送信（投稿、レビュー）
- ステータス更新（受諾、受領OK）
- 画像アップロード

**必要な機能:**

- フォームデータのバリデーション
- Supabaseクライアントを使用したDB操作
- エラーハンドリングとユーザーへのフィードバック

---

## 🔄 リアルタイム通信設計

### Supabase Realtime活用箇所

| 機能               | 用途                             |
| ------------------ | -------------------------------- |
| **チャット**       | 新規メッセージのリアルタイム受信 |
| **タイムライン**   | 新規投稿の即時表示               |
| **ステータス変更** | 依頼ステータス変更の通知         |
| **マッチング**     | 受諾通知のリアルタイム配信       |

**必要な実装:**

- Supabase Realtimeチャネルの作成
- PostgreSQLの変更イベント（INSERT, UPDATE）の購読
- フィルタ条件の設定（特定のrequest_idなど）
- ペイロードの処理とUI更新
- クリーンアップ処理（チャネルの削除）

---

## 📸 画像管理設計

### Supabase Storage構成

```
okaimono-storage/
├── profiles/           # プロフィール写真
│   └── {user_id}/
│       └── avatar.jpg
├── products/           # 商品参考写真
│   └── {request_id}/
│       └── {image_id}.jpg
├── receipts/           # レシート写真
│   └── {request_id}/
│       └── receipt.jpg
└── shelf-photos/       # 棚写真
    └── {request_id}/
        └── shelf_{timestamp}.jpg
```

### アップロードフロー

1. クライアントでファイル選択
2. Server Actionで署名付きURL生成
3. クライアントから直接Supabase Storageへアップロード
4. アップロード完了後、公開URLをDBに保存

### 画像最適化

- Next.js `<Image>` Componentで自動最適化
- Supabase Storage ImageTransformation機能でサムネイル生成

---

## 💳 決済フロー設計

### Stripe PaymentLink方式（MVP期）

```
[依頼者] 投稿時にStripe PaymentLink生成
         ↓
[代行者] 完了時にPaymentLink受領
         ↓
[代行者] リンククリック→Stripe決済画面
         ↓
[Stripe] Webhook → Next.js API Route
         ↓
[システム] payment_status更新（paid）
```

**メリット:**

- 運営が代金を預からない（資金移動法回避）
- Stripe側で決済完了管理
- 実装が簡単

**必要な機能:**

- Stripe PaymentLinkの生成
- 謝礼金額とメタデータの設定
- 生成されたURLの返却
- Webhook経由での決済完了通知の受信
- payment_statusの更新

---

## 🔔 通知設計

### MVP期の通知方式

| イベント       | 通知方法             |
| -------------- | -------------------- |
| マッチング成立 | アプリ内通知＋メール |
| 新規メッセージ | アプリ内通知         |
| ステータス変更 | アプリ内通知         |
| レビュー投稿   | メール               |

### メール送信

- **Resend API**を使用
- トランザクションメールのみ（マーケティングメール不可）

**必要な機能:**

- Resend APIクライアントの初期化
- 送信元アドレスの設定（noreply@okaimono.app）
- 受信者、件名、本文の指定
- HTMLメールテンプレートの作成
- エラーハンドリング

---

## 🛡️ セキュリティ設計

### 主要セキュリティ対策

| 項目              | 対策                                 |
| ----------------- | ------------------------------------ |
| **認証**          | Supabase Auth（OAuth 2.0）           |
| **認可**          | Row Level Security（RLS）            |
| **CSRF**          | Next.js自動保護                      |
| **XSS**           | Reactの自動エスケープ                |
| **SQL Injection** | Supabaseクライアントの自動エスケープ |
| **画像EXIF**      | アップロード時にメタデータ削除       |
| **レート制限**    | Vercel Edge Middleware               |

### RLSポリシー設計原則

1. 各テーブルにRLS有効化
2. 自分のデータまたは公開データのみアクセス可能
3. 更新・削除は所有者のみ
4. 管理者ロールも設定（将来）

---

## 📊 パフォーマンス最適化

### フロントエンド最適化

- Server Componentsで初回ロード高速化
- 動的インポート（Code Splitting）
- 画像遅延読み込み（Next.js Image）
- SWRでクライアント側キャッシング

### バックエンド最適化

- PostgreSQLインデックス設計
- Supabase Edgeネットワーク活用
- RLS最適化（複雑なクエリ回避）

### 監視・分析（将来）

- Vercel Analytics
- Sentry（エラートラッキング）
- Supabase Logs

---

## 🧪 テスト戦略

### MVP期のテスト方針

- **E2Eテスト優先**（Playwright）
  - 投稿→受諾→完了→レビューの主要フロー
- **単体テスト**は最小限（ユーティリティ関数のみ）
- **ユーザーテスト**重視（10名 × 2週間）

### テスト構成

```
tests/
├── e2e/
│   ├── request-flow.spec.ts      # 投稿→受諾フロー
│   ├── chat.spec.ts              # チャット機能
│   └── handover.spec.ts          # 受け渡し
└── unit/
    └── utils/
        └── validation.test.ts
```

---

## 🚀 デプロイ戦略

### 環境構成

| 環境            | 用途         | URL                          |
| --------------- | ------------ | ---------------------------- |
| **Development** | ローカル開発 | http://localhost:3000        |
| **Staging**     | 実証実験     | https://staging.okaimono.app |
| **Production**  | 本番（将来） | https://okaimono.app         |

### CI/CDパイプライン

```
GitHub Push
    ↓
GitHub Actions
    ├─ TypeScript型チェック
    ├─ ESLint
    ├─ Prettier
    └─ E2Eテスト（Staging）
    ↓
Vercel Auto Deploy
    ├─ Preview（PR）
    └─ Staging/Production（main）
```

### 環境変数管理

```bash
# .env.example
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
RESEND_API_KEY=
```

---

## 📈 スケーラビリティ考慮

### 現時点での設計判断

- **Supabase無料枠**: 月間50GB転送、500MBストレージ（MVP十分）
- **Vercel無料枠**: 100GB帯域（実証実験十分）
- **PostgreSQL接続プール**: Supabaseが自動管理

### 将来のスケール時対応

- Supabase Pro（$25/月）へアップグレード
- CDN追加（Cloudflare）
- Read Replica追加（読み取り負荷分散）
- Edge Functions追加（重い処理の分離）

---

## 💴 税務コンプライアンス機能

### 年間取引額の自動集計

**必要なデータ:**
| カラム | 型 | 説明 |
|--------|-----|------|
| annual_earnings_current_year | number | 今年の累計収入 |
| annual_transaction_count | number | 今年の取引回数 |
| last_earning_reset | Date | 最終リセット日 |
| tax_notification_sent | boolean | 通知送信済みフラグ |

**必要な処理:**

- 取引完了時に自動加算
- 5万円到達時に通知送信
- 20万円到達時に確定申告リマインダー（会社員向け）
- 毎年1月1日に自動リセット

### 税務通知システム

| 通知タイプ              | 発火条件                 | 配信方法         |
| ----------------------- | ------------------------ | ---------------- |
| `tax_threshold_50k`     | 年間5万円到達            | アプリ内＋メール |
| `tax_threshold_200k`    | 年間20万円到達（会社員） | アプリ内＋メール |
| `tax_year_end_reminder` | 年度末（1月末）          | メール           |

### 年間取引レポート

**APIエンドポイント:**

- `GET /api/tax-report?year=2025&format=pdf`
- `GET /api/tax-report?year=2025&format=csv`

**レポート内容:**

- 取引日時・依頼者名・謝礼金額
- 年間合計金額・取引回数
- 確定申告用の補足情報

### 支払調書管理（β版以降）

**対象:** 年間5万円超のユーザー

**処理フロー:**

1. 毎年1月中旬に自動生成
2. ユーザーにPDF送付
3. マイナンバー収集（未収集の場合）
4. 税務署への提出（1/31期限）

**必要な処理:**

- 年間5万円超のユーザー抽出
- 支払調書PDF生成
- メール送付
- マイナンバー未取得の場合は収集依頼

### マイナンバー管理

**セキュリティ要件:**

- AES-256-CBC暗号化
- アクセス権限を管理者のみに制限
- アクセスログ記録
- 特定個人情報保護法準拠

**保存期間:** 支払調書提出後7年間

---

## 次ステップ

- [データモデル設計書](./data-model.md)を参照
- [実装ロードマップ](./implementation-roadmap.md)を参照
- [税務コンプライアンス設計書](./tax-compliance.md)を参照
