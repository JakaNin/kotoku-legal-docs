# データモデル設計書

## 🗃️ ER図

```
┌─────────────┐
│   users     │
└──────┬──────┘
       │ 1
       │
       │ N
┌──────▼──────────┐        ┌──────────────┐
│   requests      │◄───1:N─┤request_items │
└──────┬──────────┘        └──────────────┘
       │ 1
       ├──────────────┐
       │              │
       │ N            │ N
┌──────▼──────┐ ┌────▼─────────┐
│  messages   │ │  handovers   │
└─────────────┘ └──────────────┘

┌─────────────┐
│   reviews   │  (users M:N users)
└─────────────┘

┌─────────────┐
│notifications│
└─────────────┘

┌─────────────┐
│ prohibited_ │
│   items     │
└─────────────┘

┌─────────────┐
│   reports   │  (報告・モデレーション)
└─────────────┘
```

---

## 📋 テーブル定義

### 1. users（ユーザー）

| カラム名                     | 型           | 制約                 | 説明                           |
| ---------------------------- | ------------ | -------------------- | ------------------------------ |
| id                           | uuid         | PK                   | Supabase Auth UUID             |
| email                        | text         | NOT NULL, UNIQUE     | メールアドレス                 |
| display_name                 | text         | NOT NULL             | 表示名（実名）                 |
| avatar_url                   | text         |                      | プロフィール画像URL            |
| address_prefecture           | text         | NOT NULL             | 都道府県                       |
| address_city                 | text         | NOT NULL             | 市区町村                       |
| address_area                 | text         | NOT NULL             | 町丁目（公開情報）             |
| reputation_score             | numeric(3,2) | DEFAULT 0            | 平均評価スコア（0-5）          |
| completed_requests_count     | int          | DEFAULT 0            | 完了した依頼数                 |
| completed_errands_count      | int          | DEFAULT 0            | 完了した代行数                 |
| annual_earnings_current_year | int          | DEFAULT 0            | 今年の累計収入                 |
| annual_transaction_count     | int          | DEFAULT 0            | 今年の取引回数                 |
| last_earning_reset           | date         | DEFAULT CURRENT_DATE | 最終リセット日                 |
| tax_notification_sent        | boolean      | DEFAULT false        | 税務通知送信済み               |
| role                         | text         | DEFAULT 'user'       | ユーザー権限（user/admin）     |
| is_banned                    | boolean      | DEFAULT false        | BANフラグ（Phase 2で使用）     |
| banned_at                    | timestamptz  |                      | BAN実行日時（Phase 2で使用）   |
| ban_reason                   | text         |                      | BAN理由（Phase 2で使用）       |
| banned_by                    | uuid         | FK(users)            | BAN実行管理者（Phase 2で使用） |
| created_at                   | timestamptz  | DEFAULT now()        | 登録日時                       |
| updated_at                   | timestamptz  | DEFAULT now()        | 更新日時                       |

**インデックス:**

```sql
CREATE INDEX idx_users_area ON users(address_area);
CREATE INDEX idx_users_email ON users(email);
```

**RLS:**

```sql
-- 自分のプロフィールのみ更新可能
CREATE POLICY "Users can update own profile"
ON users FOR UPDATE
USING (auth.uid() = id);

-- 全員が全ユーザーを閲覧可能（公開情報のみ）
CREATE POLICY "Users are viewable by everyone"
ON users FOR SELECT
USING (true);
```

---

### 2. requests（買い物依頼）

| カラム名                  | 型          | 制約                       | 説明                   |
| ------------------------- | ----------- | -------------------------- | ---------------------- |
| id                        | uuid        | PK                         | 依頼ID                 |
| user_id                   | uuid        | FK(users), NOT NULL        | 依頼者ID               |
| title                     | text        | NOT NULL                   | 依頼タイトル           |
| description               | text        |                            | 補足説明               |
| reward_amount             | int         | NOT NULL, DEFAULT 500      | 謝礼金額（円）         |
| total_price_cap           | int         |                            | 合計上限金額           |
| preferred_pickup_date     | date        |                            | 希望受け取り日         |
| preferred_pickup_time     | text        |                            | 希望受け取り時間帯     |
| address_area              | text        | NOT NULL                   | 依頼者エリア（町丁目） |
| status                    | text        | NOT NULL, DEFAULT 'posted' | ステータス             |
| matched_user_id           | uuid        | FK(users)                  | 受諾者ID               |
| matched_at                | timestamptz |                            | 受諾日時               |
| completed_at              | timestamptz |                            | 完了日時               |
| payment_link_url          | text        |                            | Stripe PaymentLink URL |
| payment_status            | text        | DEFAULT 'unpaid'           | 決済ステータス         |
| receipt_original_required | boolean     | DEFAULT false              | レシート原本が必要か   |
| created_at                | timestamptz | DEFAULT now()              | 投稿日時               |
| updated_at                | timestamptz | DEFAULT now()              | 更新日時               |

**ステータス値:**

- `posted`: 投稿済み（未マッチング）
- `matched`: マッチング成立
- `shopping`: 買い物中
- `awaiting_approval`: 棚写真承認待ち
- `purchased`: 購入完了
- `delivering`: 受け渡し中
- `completed`: 完了
- `reviewed`: レビュー済み
- `cancelled`: キャンセル

**インデックス:**

```sql
CREATE INDEX idx_requests_status ON requests(status);
CREATE INDEX idx_requests_area ON requests(address_area);
CREATE INDEX idx_requests_user_id ON requests(user_id);
CREATE INDEX idx_requests_matched_user_id ON requests(matched_user_id);
CREATE INDEX idx_requests_created_at ON requests(created_at DESC);
```

**RLS:**

```sql
-- 自分のエリアの投稿を閲覧可能
CREATE POLICY "Users can view requests in their area"
ON requests FOR SELECT
USING (
  address_area = (SELECT address_area FROM users WHERE id = auth.uid())
  OR user_id = auth.uid()
  OR matched_user_id = auth.uid()
);

-- 自分の投稿のみ作成・更新可能
CREATE POLICY "Users can create own requests"
ON requests FOR INSERT
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own requests"
ON requests FOR UPDATE
USING (auth.uid() = user_id OR auth.uid() = matched_user_id);
```

---

### 3. request_items（依頼商品明細）

| カラム名            | 型          | 制約                   | 説明                 |
| ------------------- | ----------- | ---------------------- | -------------------- |
| id                  | uuid        | PK                     | 商品明細ID           |
| request_id          | uuid        | FK(requests), NOT NULL | 依頼ID               |
| item_name           | text        | NOT NULL               | 商品名               |
| quantity            | int         | NOT NULL, DEFAULT 1    | 数量                 |
| price_cap           | int         |                        | 上限価格（円）       |
| substitute_allowed  | boolean     | DEFAULT false          | 代替品可否           |
| substitute_note     | text        |                        | 代替品に関する注意   |
| reference_image_url | text        |                        | 商品参考画像URL      |
| actual_price        | int         |                        | 実際の購入価格       |
| purchased_item_name | text        |                        | 実際に購入した商品名 |
| created_at          | timestamptz | DEFAULT now()          | 作成日時             |

**インデックス:**

```sql
CREATE INDEX idx_request_items_request_id ON request_items(request_id);
```

**RLS:**

```sql
-- request_itemsはrequestsと同じポリシー
CREATE POLICY "Users can view items of accessible requests"
ON request_items FOR SELECT
USING (
  request_id IN (
    SELECT id FROM requests WHERE
      address_area = (SELECT address_area FROM users WHERE id = auth.uid())
      OR user_id = auth.uid()
      OR matched_user_id = auth.uid()
  )
);
```

---

### 4. messages（チャット）

| カラム名     | 型          | 制約                   | 説明                |
| ------------ | ----------- | ---------------------- | ------------------- |
| id           | uuid        | PK                     | メッセージID        |
| request_id   | uuid        | FK(requests), NOT NULL | 依頼ID              |
| sender_id    | uuid        | FK(users), NOT NULL    | 送信者ID            |
| content      | text        |                        | メッセージ本文      |
| image_url    | text        |                        | 画像URL（棚写真等） |
| message_type | text        | DEFAULT 'text'         | メッセージタイプ    |
| read_at      | timestamptz |                        | 既読日時            |
| created_at   | timestamptz | DEFAULT now()          | 送信日時            |

**メッセージタイプ:**

- `text`: テキストメッセージ
- `image`: 画像メッセージ
- `shelf_photo`: 棚写真（購入確認用）
- `system`: システムメッセージ

**インデックス:**

```sql
CREATE INDEX idx_messages_request_id ON messages(request_id);
CREATE INDEX idx_messages_created_at ON messages(created_at DESC);
```

**RLS:**

```sql
-- 依頼の当事者のみ閲覧可能
CREATE POLICY "Request participants can view messages"
ON messages FOR SELECT
USING (
  request_id IN (
    SELECT id FROM requests
    WHERE user_id = auth.uid() OR matched_user_id = auth.uid()
  )
);

CREATE POLICY "Request participants can send messages"
ON messages FOR INSERT
WITH CHECK (
  sender_id = auth.uid()
  AND request_id IN (
    SELECT id FROM requests
    WHERE user_id = auth.uid() OR matched_user_id = auth.uid()
  )
);
```

---

### 5. handovers（受け渡し記録）

| カラム名                   | 型          | 制約                           | 説明                   |
| -------------------------- | ----------- | ------------------------------ | ---------------------- |
| id                         | uuid        | PK                             | 受け渡しID             |
| request_id                 | uuid        | FK(requests), NOT NULL, UNIQUE | 依頼ID                 |
| receipt_image_url          | text        |                                | レシート画像URL        |
| product_images             | text[]      |                                | 商品画像URL配列        |
| total_amount               | int         |                                | 合計金額               |
| is_sealed                  | boolean     | DEFAULT true                   | 未開封確認             |
| confirmed_by_requester     | boolean     | DEFAULT false                  | 依頼者確認済み         |
| confirmed_at               | timestamptz |                                | 確認日時               |
| receipt_original_delivered | boolean     | DEFAULT false                  | レシート原本を渡したか |
| notes                      | text        |                                | 備考                   |
| created_at                 | timestamptz | DEFAULT now()                  | 作成日時               |

**レシート管理方針:**

- レシート原本の取り扱いについては [`docs/implementation/receipt-management-policy.md`](./implementation/receipt-management-policy.md) を参照
- MVP期は依頼者が選択（`requests.receipt_original_required`）し、必要な場合のみ代行者がレシート原本を渡す
- デフォルトはデジタル画像のみ（`receipt_original_required = false`）

**インデックス:**

```sql
CREATE INDEX idx_handovers_request_id ON handovers(request_id);
```

**RLS:**

```sql
-- 依頼の当事者のみ閲覧・更新可能
CREATE POLICY "Request participants can manage handovers"
ON handovers FOR ALL
USING (
  request_id IN (
    SELECT id FROM requests
    WHERE user_id = auth.uid() OR matched_user_id = auth.uid()
  )
);
```

---

### 6. reviews（レビュー）

| カラム名     | 型          | 制約                   | 説明          |
| ------------ | ----------- | ---------------------- | ------------- |
| id           | uuid        | PK                     | レビューID    |
| request_id   | uuid        | FK(requests), NOT NULL | 依頼ID        |
| from_user_id | uuid        | FK(users), NOT NULL    | 評価者ID      |
| to_user_id   | uuid        | FK(users), NOT NULL    | 被評価者ID    |
| rating       | int         | NOT NULL, CHECK(1-5)   | 星評価（1-5） |
| comment      | text        |                        | コメント      |
| created_at   | timestamptz | DEFAULT now()          | 投稿日時      |

**制約:**

```sql
-- 同じ依頼で同じユーザーが2回評価できない
CREATE UNIQUE INDEX idx_reviews_unique ON reviews(request_id, from_user_id);
```

**インデックス:**

```sql
CREATE INDEX idx_reviews_to_user_id ON reviews(to_user_id);
CREATE INDEX idx_reviews_request_id ON reviews(request_id);
```

**RLS:**

```sql
-- 全員がレビューを閲覧可能
CREATE POLICY "Reviews are public"
ON reviews FOR SELECT
USING (true);

-- 依頼の当事者のみレビュー投稿可能
CREATE POLICY "Request participants can create reviews"
ON reviews FOR INSERT
WITH CHECK (
  from_user_id = auth.uid()
  AND request_id IN (
    SELECT id FROM requests
    WHERE user_id = auth.uid() OR matched_user_id = auth.uid()
  )
);
```

---

### 7. notifications（通知）

| カラム名   | 型          | 制約                | 説明        |
| ---------- | ----------- | ------------------- | ----------- |
| id         | uuid        | PK                  | 通知ID      |
| user_id    | uuid        | FK(users), NOT NULL | 受信者ID    |
| type       | text        | NOT NULL            | 通知タイプ  |
| title      | text        | NOT NULL            | タイトル    |
| content    | text        |                     | 内容        |
| link_url   | text        |                     | リンク先URL |
| read_at    | timestamptz |                     | 既読日時    |
| created_at | timestamptz | DEFAULT now()       | 作成日時    |

**通知タイプ:**

- `match`: マッチング成立
- `message`: 新規メッセージ
- `status_change`: ステータス変更
- `review`: レビュー投稿
- `payment`: 決済完了
- `tax_threshold_50k`: 年間5万円到達
- `tax_threshold_200k`: 年間20万円到達
- `tax_year_end_reminder`: 年度末確定申告リマインダー

**インデックス:**

```sql
CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_created_at ON notifications(created_at DESC);
CREATE INDEX idx_notifications_read_at ON notifications(read_at);
```

**RLS:**

```sql
-- 自分の通知のみ閲覧・更新可能
CREATE POLICY "Users can manage own notifications"
ON notifications FOR ALL
USING (auth.uid() = user_id);
```

---

### 8. prohibited_items（禁止品目マスタ）

| カラム名   | 型          | 制約             | 説明           |
| ---------- | ----------- | ---------------- | -------------- |
| id         | uuid        | PK               | 禁止品目ID     |
| keyword    | text        | NOT NULL, UNIQUE | 禁止キーワード |
| category   | text        | NOT NULL         | カテゴリ       |
| reason     | text        |                  | 禁止理由       |
| is_active  | boolean     | DEFAULT true     | 有効フラグ     |
| created_at | timestamptz | DEFAULT now()    | 作成日時       |

**カテゴリ:**

- `alcohol`: 酒類
- `medicine`: 医薬品
- `tobacco`: たばこ
- `cosmetics`: 化粧品
- `perishable`: 生鮮食品
- `cash`: 現金・金券
- `high_value`: 高額商品
- `living`: 動植物

**初期データ:**

```sql
INSERT INTO prohibited_items (keyword, category, reason) VALUES
('酒', 'alcohol', '酒類販売は許認可が必要'),
('ビール', 'alcohol', '酒類販売は許認可が必要'),
('ワイン', 'alcohol', '酒類販売は許認可が必要'),
('医薬品', 'medicine', '医薬品販売は許認可が必要'),
('薬', 'medicine', '医薬品販売は許認可が必要'),
('たばこ', 'tobacco', 'たばこ販売は許認可が必要'),
('化粧品', 'cosmetics', '化粧品は品質保証困難'),
('肉', 'perishable', '生鮮食品は品質保証困難'),
('魚', 'perishable', '生鮮食品は品質保証困難'),
('商品券', 'cash', '現金・金券は不可');
```

**RLS:**

```sql
-- 全員が閲覧可能
CREATE POLICY "Prohibited items are public"
ON prohibited_items FOR SELECT
USING (true);
```

---

### 9. reports（報告・モデレーション）

| カラム名             | 型          | 制約                | 説明                          |
| -------------------- | ----------- | ------------------- | ----------------------------- |
| id                   | uuid        | PK                  | 報告ID                        |
| reporter_id          | uuid        | FK(users), NOT NULL | 報告者ID                      |
| report_type          | text        | NOT NULL            | 報告対象の種類                |
| reported_user_id     | uuid        | FK(users)           | 報告されたユーザーID          |
| reported_request_id  | uuid        | FK(requests)        | 報告された依頼ID              |
| reported_message_id  | uuid        | FK(messages)        | 報告されたメッセージID        |
| reported_handover_id | uuid        | FK(handovers)       | 報告された受け渡しID          |
| reason               | text        | NOT NULL            | 報告理由カテゴリ              |
| description          | text        | NOT NULL            | 詳細説明                      |
| status               | text        | DEFAULT 'pending'   | 報告ステータス                |
| admin_notes          | text        |                     | 管理者メモ（Phase 2で使用）   |
| resolved_by          | uuid        | FK(users)           | 対応管理者ID（Phase 2で使用） |
| resolved_at          | timestamptz |                     | 対応完了日時（Phase 2で使用） |
| created_at           | timestamptz | DEFAULT now()       | 報告日時                      |
| updated_at           | timestamptz | DEFAULT now()       | 更新日時                      |

**報告対象タイプ（report_type）:**

- `user`: ユーザー
- `request`: 依頼
- `message`: メッセージ
- `handover`: 受け渡し

**報告理由カテゴリ（reason）:**

- `prohibited_items`: 禁止品目
- `harassment`: ハラスメント・誹謗中傷
- `fraud`: 詐欺・詐欺未遂
- `inappropriate_content`: 不適切な内容
- `spam`: スパム・宣伝
- `fake_profile`: 偽プロフィール・なりすまし
- `payment_issue`: 支払いトラブル
- `other`: その他

**ステータス（status）:**

- `pending`: 未処理
- `reviewing`: 確認中（Phase 2で使用）
- `resolved`: 対応完了（Phase 2で使用）
- `rejected`: 却下（Phase 2で使用）

**制約:**

```sql
-- 報告対象は1つだけ（いずれか1つのみNOT NULL）
ALTER TABLE reports ADD CONSTRAINT report_target_check CHECK (
  (reported_user_id IS NOT NULL)::int +
  (reported_request_id IS NOT NULL)::int +
  (reported_message_id IS NOT NULL)::int +
  (reported_handover_id IS NOT NULL)::int = 1
);
```

**インデックス:**

```sql
CREATE INDEX idx_reports_status ON reports(status) WHERE status = 'pending';
CREATE INDEX idx_reports_created_at ON reports(created_at DESC);
CREATE INDEX idx_reports_reporter ON reports(reporter_id);
CREATE INDEX idx_reports_reported_user ON reports(reported_user_id) WHERE reported_user_id IS NOT NULL;
```

**RLS:**

```sql
-- 一般ユーザーは自分の報告のみ閲覧可能
CREATE POLICY "Users can view their own reports"
  ON reports FOR SELECT
  TO authenticated
  USING (reporter_id = auth.uid());

-- 一般ユーザーは報告作成可能（BANされていない場合のみ）
CREATE POLICY "Users can create reports"
  ON reports FOR INSERT
  TO authenticated
  WITH CHECK (
    reporter_id = auth.uid()
    AND EXISTS (
      SELECT 1 FROM users
      WHERE id = auth.uid()
      AND is_banned = FALSE
    )
  );

-- Phase 2: 管理者のみ報告ステータス更新・管理者メモ追加可能（実装時に有効化）
-- CREATE POLICY "Admins can update reports"
--   ON reports FOR UPDATE
--   TO authenticated
--   USING (
--     EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin')
--   );
```

**Phase 1 運用方法:**

- 報告はデータベースに蓄積
- 管理者はSupabase Studioで直接確認
- BAN等の対応は手動でSQLを実行

**Phase 2 実装予定:**

- 管理者ダッシュボード（`/admin`）
- 報告詳細ページ
- BAN実行・解除UI
- BANユーザーの制限（RLSポリシー）

**詳細ドキュメント:**

- [`tmp/moderation-system-design.md`](moderation-system-design.md) - 報告・モデレーション機能の詳細設計

---

### 10. payment_reports（支払調書）

| カラム名                | 型          | 制約                | 説明                 |
| ----------------------- | ----------- | ------------------- | -------------------- |
| id                      | uuid        | PK                  | 支払調書ID           |
| user_id                 | uuid        | FK(users), NOT NULL | 受取人ID             |
| year                    | int         | NOT NULL            | 対象年度             |
| total_amount            | int         | NOT NULL            | 年間合計金額         |
| transaction_count       | int         | NOT NULL            | 取引回数             |
| report_pdf_url          | text        |                     | 支払調書PDF URL      |
| mynumber_collected      | boolean     | DEFAULT false       | マイナンバー取得済み |
| submitted_to_tax_office | boolean     | DEFAULT false       | 税務署提出済み       |
| submitted_at            | timestamptz |                     | 提出日時             |
| created_at              | timestamptz | DEFAULT now()       | 作成日時             |

**制約:**

```sql
CREATE UNIQUE INDEX idx_payment_reports_unique ON payment_reports(user_id, year);
```

**インデックス:**

```sql
CREATE INDEX idx_payment_reports_user_id ON payment_reports(user_id);
CREATE INDEX idx_payment_reports_year ON payment_reports(year);
```

**RLS:**

```sql
-- 自分の支払調書のみ閲覧可能
CREATE POLICY "Users can view own payment reports"
ON payment_reports FOR SELECT
USING (auth.uid() = user_id);
```

---

### 11. user_mynumbers（マイナンバー管理）

| カラム名           | 型          | 制約          | 説明                     |
| ------------------ | ----------- | ------------- | ------------------------ |
| user_id            | uuid        | PK, FK(users) | ユーザーID               |
| mynumber_encrypted | text        | NOT NULL      | 暗号化されたマイナンバー |
| collected_at       | timestamptz | DEFAULT now() | 取得日時                 |
| consent_given      | boolean     | DEFAULT false | 利用同意取得済み         |
| created_at         | timestamptz | DEFAULT now() | 作成日時                 |

**セキュリティ:**

- AES-256-CBC暗号化必須
- アクセスは管理者のみに制限
- アクセスログ記録必須

**RLS:**

```sql
-- 管理者のみアクセス可能
CREATE POLICY "Only admin can access mynumbers"
ON user_mynumbers FOR ALL
USING (
  auth.uid() IN (SELECT id FROM admin_users WHERE role IN ('super_admin', 'tax_admin'))
);
```

---

## 🔧 必要なインデックス

### パフォーマンス最適化のためのインデックス

**users テーブル:**

- `address_area`: エリア別検索用
- `email`: メールアドレス検索用

**requests テーブル:**

- `status`: ステータス別フィルタ用
- `address_area`: エリア別検索用
- `user_id`: ユーザーの依頼一覧用
- `matched_user_id`: 受諾者の履歴用
- `created_at`: 新着順ソート用（降順）

**その他のテーブル:**

- `request_items.request_id`: 依頼の商品一覧取得用
- `messages.request_id`: チャット履歴取得用
- `messages.created_at`: メッセージ時系列ソート用（降順）
- `reviews.to_user_id`: ユーザー評価一覧用
- `reviews.(request_id, from_user_id)`: 重複レビュー防止用（UNIQUE）
- `notifications.user_id`: 通知一覧取得用
- `notifications.created_at`: 通知時系列ソート用（降順）

### Row Level Security (RLS)

**すべてのテーブルでRLSを有効化:**

- 詳細なRLSポリシーは各テーブル定義のセクションを参照

---

## 🔄 必要なトリガー・関数

### 1. reputation_score自動更新

**目的:** レビュー投稿時にユーザーの評価スコアを自動更新

**トリガー条件:** `reviews`テーブルへのINSERT

**処理内容:**

- 被評価者（to_user_id）の全レビューの平均を計算
- usersテーブルのreputation_scoreを更新

### 2. 完了カウント更新

**目的:** 依頼完了時に完了数を自動インクリメント

**トリガー条件:** `requests`テーブルのUPDATE（status が completed に変更）

**処理内容:**

- 依頼者のcompleted_requests_countを+1
- 代行者のcompleted_errands_countを+1

### 3. updated_at自動更新

**目的:** レコード更新時にupdated_atを自動更新

**トリガー条件:** `users`, `requests`テーブルのUPDATE

**処理内容:**

- updated_atカラムを現在時刻に更新

---

## 📈 想定データ量（MVP期）

| テーブル         | 想定レコード数 | 備考                      |
| ---------------- | -------------- | ------------------------- |
| users            | 10-50          | テストユーザー            |
| requests         | 10-30          | 1ヶ月で10-30件            |
| request_items    | 30-100         | 依頼あたり3-5商品         |
| messages         | 100-500        | 依頼あたり10-20メッセージ |
| handovers        | 5-15           | 完了した取引のみ          |
| reviews          | 10-30          | 完了取引の相互評価        |
| notifications    | 100-300        | 各アクションで生成        |
| prohibited_items | 50-100         | 禁止キーワードマスタ      |
| reports          | 0-10           | MVP期は少数の報告を想定   |

---

## 次ステップ

- [実装ロードマップ](./implementation-roadmap.md)を参照してマイグレーション実行計画を確認
