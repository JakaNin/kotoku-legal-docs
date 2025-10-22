# 報告・モデレーション機能 設計書

**作成日**: 2025-10-12
**ステータス**: レビュー中（Draft）
**対象**: Kotoku プラットフォーム

---

## 📋 目次

1. [概要](#概要)
2. [実装方針（重要）](#実装方針重要)
3. [要件定義](#要件定義)
4. [データベース設計](#データベース設計)
5. [BAN機能設計（Phase 2・TODO）](#ban機能設計phase-2todo)
6. [管理者権限設計（Phase 2・TODO）](#管理者権限設計phase-2todo)
7. [RLSポリシー](#rlsポリシー)
8. [ユースケース](#ユースケース)
9. [実装計画](#実装計画)

---

## 概要

### 目的

- **安全なプラットフォーム運営**: 不適切なユーザー・コンテンツの報告とモデレーション
- **詐欺・ハラスメント防止**: 悪質なユーザーの迅速なBAN
- **コミュニティの信頼性向上**: 透明性のある報告・対応プロセス

### スコープ（全体）

- ✅ 報告機能（ユーザー、依頼、メッセージ、受け渡し）
- 📝 BAN機能（ユーザー単位）- **Phase 2で実装**
- 📝 管理者権限（報告確認・BAN実行）- **Phase 2で実装**
- ⏳ 自動モデレーション（将来実装）
- ⏳ 通報者保護機能（匿名化）

---

## 実装方針（重要）

### Phase 1（MVP・今回実装）: 報告機能のみ

**実装範囲**:

- ✅ `reports` テーブル作成
- ✅ 報告UI（ユーザー、依頼、メッセージ、受け渡しを報告可能）
- ✅ RLSポリシー（一般ユーザーは自分の報告のみ閲覧）
- ✅ `users` テーブルにBAN関連カラム追加（**スキーマのみ、使用はPhase 2**）

**運用方法**:

- 報告はデータベースに蓄積されるのみ
- 管理者はSupabase Studioで直接データベースを確認
- BAN等の対応は手動（SQLで `is_banned = TRUE` 等を設定）

**メリット**:

- 実装工数が少ない（報告UI + テーブルのみ）
- MVP期の小規模運用に適している
- Phase 2への移行が容易（スキーマは準備済み）

### Phase 2（TODO・将来実装）: 管理者UI + BAN機能

**実装範囲**:

- 管理者ダッシュボード（`/admin`）
- 報告詳細ページ
- BAN実行・解除UI
- BANユーザーの制限（RLSポリシーで投稿・受諾・メッセージ禁止）
- 通知機能（BAN時にユーザーへ通知）

**タイミング**:

- MVP実証実験後
- ユーザー数増加時（報告が増えて手動対応が困難になった時点）

---

### 実装判断の理由

**なぜPhase 1で管理者UIを作らないのか？**:

1. **MVP期は報告数が少ない**
   - 実証実験の参加者は限定的（10-20人程度）
   - 報告は1日1件あるかないか程度と想定
   - Supabase Studioで十分対応可能

2. **開発工数の削減**
   - 管理者UI実装には4-6時間必要
   - その時間をコア機能（依頼投稿、チャット等）に充てたい

3. **要件の不確実性**
   - 実証実験でどんな報告が来るか不明
   - 実際の運用を見てから管理UIを設計した方が効率的

4. **段階的な機能拡張**
   - スキーマは今回準備するので、Phase 2への移行は容易
   - データは蓄積されているので、後から分析・対応可能

---

## 要件定義

### 機能要件

#### FR-1: 報告機能

**誰が**: 全ての認証済みユーザー
**何を**: 以下4種類のエンティティを報告可能

- ユーザー（プロフィール、行動全般）
- 依頼（リクエスト）
- メッセージ（チャット）
- 受け渡し（ハンドオーバー）

**報告理由カテゴリ**:
| カテゴリ | 説明 | 対象 |
|---------|------|------|
| `prohibited_items` | 禁止品目の依頼 | リクエスト |
| `harassment` | ハラスメント・誹謗中傷 | ユーザー、メッセージ |
| `fraud` | 詐欺・詐欺未遂 | ユーザー、リクエスト、ハンドオーバー |
| `inappropriate_content` | 不適切な内容（性的・暴力的） | ユーザー、リクエスト、メッセージ |
| `spam` | スパム・宣伝 | ユーザー、リクエスト、メッセージ |
| `fake_profile` | 偽プロフィール・なりすまし | ユーザー |
| `payment_issue` | 支払いトラブル | ハンドオーバー |
| `other` | その他 | 全て |

**必須情報**:

- 報告理由（カテゴリ選択）
- 詳細説明（自由記述、最低20文字）

**任意情報**:

- 証拠画像（将来実装）

#### FR-2: BAN機能

**誰が**: 管理者（role='admin'）のみ
**何を**: 悪質なユーザーをBANし、プラットフォーム利用を制限

**BANされたユーザーの制限**:

- ❌ 新規依頼投稿
- ❌ 依頼受諾
- ❌ メッセージ送信
- ✅ ログイン可能（閲覧のみ、BAN通知表示）
- ✅ 過去の取引履歴閲覧

**BAN解除**: 管理者が手動で解除可能

#### FR-3: 管理者機能

**管理者権限**:

- 報告一覧閲覧
- 報告詳細確認
- 報告ステータス変更（pending → reviewing → resolved/rejected）
- ユーザーBAN/BAN解除
- BAN理由記録
- 管理者メモ追加

**管理者ダッシュボード** (将来実装):

- 未処理報告数
- BAN済みユーザー数
- 報告カテゴリ別統計

### 非機能要件

#### NFR-1: セキュリティ

- **RLS必須**: 一般ユーザーは自分の報告のみ閲覧可能
- **管理者認証**: role='admin'の検証必須
- **監査ログ**: BAN実行者・日時を記録

#### NFR-2: プライバシー

- **報告者保護**: 報告者情報は被報告者に非公開（管理者のみ閲覧可）
- **削除ポリシー**: ユーザー削除時、報告も連鎖削除（CASCADE）

#### NFR-3: パフォーマンス

- **インデックス**: status, created_at, reporter_id, reported_user_id
- **ページネーション**: 管理者ダッシュボードで報告一覧表示

---

## データベース設計

### 1. reportsテーブル

```sql
-- 報告テーブル
CREATE TABLE reports (
  -- 基本情報
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  reporter_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  report_type TEXT NOT NULL CHECK (report_type IN (
    'user',
    'request',
    'message',
    'handover'
  )),

  -- 報告対象（いずれか1つのみNOT NULL）
  reported_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  reported_request_id UUID REFERENCES requests(id) ON DELETE CASCADE,
  reported_message_id UUID REFERENCES messages(id) ON DELETE CASCADE,
  reported_handover_id UUID REFERENCES handovers(id) ON DELETE CASCADE,

  -- 報告内容
  reason TEXT NOT NULL CHECK (reason IN (
    'prohibited_items',
    'harassment',
    'fraud',
    'inappropriate_content',
    'spam',
    'fake_profile',
    'payment_issue',
    'other'
  )),
  description TEXT NOT NULL CHECK (LENGTH(description) >= 20),

  -- ステータス管理
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
    'pending',    -- 未処理
    'reviewing',  -- 確認中
    'resolved',   -- 対応完了（BAN等実行済み）
    'rejected'    -- 却下（問題なし）
  )),

  -- 管理者対応
  admin_notes TEXT,
  resolved_by UUID REFERENCES users(id) ON DELETE SET NULL,
  resolved_at TIMESTAMPTZ,

  -- タイムスタンプ
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- 制約: 報告対象は1つだけ
  CONSTRAINT report_target_check CHECK (
    (reported_user_id IS NOT NULL)::int +
    (reported_request_id IS NOT NULL)::int +
    (reported_message_id IS NOT NULL)::int +
    (reported_handover_id IS NOT NULL)::int = 1
  )
);

-- インデックス
CREATE INDEX idx_reports_status ON reports(status) WHERE status = 'pending';
CREATE INDEX idx_reports_created_at ON reports(created_at DESC);
CREATE INDEX idx_reports_reporter ON reports(reporter_id);
CREATE INDEX idx_reports_reported_user ON reports(reported_user_id) WHERE reported_user_id IS NOT NULL;
CREATE INDEX idx_reports_type_status ON reports(report_type, status);

-- トリガー: updated_at自動更新
CREATE TRIGGER reports_updated_at
  BEFORE UPDATE ON reports
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

-- コメント
COMMENT ON TABLE reports IS '不適切なユーザー・コンテンツの報告';
COMMENT ON COLUMN reports.report_type IS '報告対象の種類（user/request/message/handover）';
COMMENT ON COLUMN reports.reason IS '報告理由カテゴリ';
COMMENT ON COLUMN reports.status IS '報告ステータス（pending/reviewing/resolved/rejected）';
COMMENT ON COLUMN reports.admin_notes IS '管理者専用メモ';
```

### 2. usersテーブル拡張

```sql
-- 既存のusersテーブルに以下のカラムを追加

ALTER TABLE users ADD COLUMN IF NOT EXISTS role TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin'));
ALTER TABLE users ADD COLUMN IF NOT EXISTS is_banned BOOLEAN NOT NULL DEFAULT FALSE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS banned_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN IF NOT EXISTS ban_reason TEXT;
ALTER TABLE users ADD COLUMN IF NOT EXISTS banned_by UUID REFERENCES users(id) ON DELETE SET NULL;

-- インデックス
CREATE INDEX idx_users_role ON users(role) WHERE role = 'admin';
CREATE INDEX idx_users_banned ON users(is_banned) WHERE is_banned = TRUE;

-- コメント
COMMENT ON COLUMN users.role IS 'ユーザー権限（user/admin）';
COMMENT ON COLUMN users.is_banned IS 'BANフラグ';
COMMENT ON COLUMN users.banned_at IS 'BAN実行日時';
COMMENT ON COLUMN users.ban_reason IS 'BAN理由';
COMMENT ON COLUMN users.banned_by IS 'BAN実行した管理者のID';
```

---

## BAN機能設計（Phase 2・TODO）

> **注意**: このセクションはPhase 2で実装予定の内容です。Phase 1ではスキーマのみ準備します。

### BANフロー

```
1. 管理者が報告を確認
   ↓
2. BANが必要と判断
   ↓
3. users.is_banned = TRUE
   users.banned_at = NOW()
   users.ban_reason = '理由'
   users.banned_by = 管理者ID
   ↓
4. 報告ステータスを 'resolved' に更新
   reports.resolved_by = 管理者ID
   reports.resolved_at = NOW()
   reports.admin_notes = '対応内容'
   ↓
5. BANされたユーザーに通知（通知テーブル経由）
```

### BAN時の制約（RLSポリシーで実装）

**requests テーブル**:

```sql
-- BANされたユーザーは新規依頼投稿不可
CREATE POLICY "Banned users cannot create requests"
  ON requests FOR INSERT
  TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM users
      WHERE id = auth.uid()
      AND is_banned = FALSE
    )
  );
```

**request_matches テーブル**:

```sql
-- BANされたユーザーは依頼受諾不可
CREATE POLICY "Banned users cannot accept requests"
  ON request_matches FOR INSERT
  TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM users
      WHERE id = auth.uid()
      AND is_banned = FALSE
    )
  );
```

**messages テーブル**:

```sql
-- BANされたユーザーはメッセージ送信不可
CREATE POLICY "Banned users cannot send messages"
  ON messages FOR INSERT
  TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM users
      WHERE id = auth.uid()
      AND is_banned = FALSE
    )
  );
```

### BAN解除フロー

```
1. 管理者が解除判断
   ↓
2. users.is_banned = FALSE
   users.ban_reason = NULL
   users.banned_by = NULL
   users.banned_at = NULL
   ↓
3. ユーザーに通知（BAN解除を通知）
```

---

## 管理者権限設計（Phase 2・TODO）

> **注意**: このセクションはPhase 2で実装予定の内容です。Phase 1では `role` カラムのみ追加します。

### 管理者の種類

**Phase 1 (MVP)**: 単一の管理者ロール

- `role = 'admin'`: 全ての管理機能にアクセス可能

**Phase 2 (将来)**: 階層的な権限

- `role = 'super_admin'`: 全権限
- `role = 'moderator'`: 報告確認・BAN実行のみ
- `role = 'support'`: 報告閲覧のみ

### 管理者の作成方法

**初期管理者**: マイグレーションで作成（開発者アカウント）

```sql
-- 初期管理者アカウント作成（手動）
-- Supabase Authで作成後、roleを更新
UPDATE users SET role = 'admin' WHERE email = 'admin@kotoku.app';
```

**追加管理者**: 既存の管理者が他ユーザーを昇格

```sql
-- 管理者のみ実行可能（RLSポリシーで制御）
UPDATE users SET role = 'admin' WHERE id = '<user_id>';
```

---

## RLSポリシー

### reportsテーブル

```sql
-- RLS有効化
ALTER TABLE reports ENABLE ROW LEVEL SECURITY;

-- 一般ユーザー: 自分が報告した内容のみ閲覧可能
CREATE POLICY "Users can view their own reports"
  ON reports FOR SELECT
  TO authenticated
  USING (
    reporter_id = auth.uid()
    OR
    EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin')
  );

-- 一般ユーザー: 報告作成可能（BANされていない場合のみ）
CREATE POLICY "Users can create reports"
  ON reports FOR INSERT
  TO authenticated
  WITH CHECK (
    reporter_id = auth.uid()
    AND
    EXISTS (
      SELECT 1 FROM users
      WHERE id = auth.uid()
      AND is_banned = FALSE
    )
  );

-- 管理者のみ: 報告ステータス更新・管理者メモ追加可能
CREATE POLICY "Admins can update reports"
  ON reports FOR UPDATE
  TO authenticated
  USING (
    EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin')
  )
  WITH CHECK (
    EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin')
  );

-- 報告の削除は不可（監査ログとして保持）
-- 削除ポリシーなし
```

### usersテーブル（BAN関連）

```sql
-- 管理者のみ: BANフラグ更新可能
CREATE POLICY "Admins can ban users"
  ON users FOR UPDATE
  TO authenticated
  USING (
    EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin')
  )
  WITH CHECK (
    EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin')
  );

-- 注意: 既存のusersテーブルのSELECTポリシーは維持
-- （全ユーザーが他のユーザーの基本情報を閲覧可能）
```

---

## ユースケース

### UC-1: ユーザーが不適切なメッセージを報告

**アクター**: 依頼者（ユーザーA）
**前提条件**: ユーザーAが代行者（ユーザーB）からハラスメントメッセージを受信

**フロー**:

1. ユーザーAがメッセージの「報告」ボタンをクリック
2. 報告フォーム表示
   - 報告理由: `harassment`
   - 詳細説明: 「何度も不適切な言葉で罵られました」
3. 報告送信
   - `reports` テーブルに新規レコード作成
   - `status = 'pending'`
   - `report_type = 'message'`
   - `reported_message_id = <message_id>`
4. 管理者に通知（通知テーブル経由）
5. ユーザーAに「報告を受け付けました」確認画面表示

**事後条件**:

- 報告が管理者ダッシュボードに表示される
- ユーザーBには報告されたことは通知されない（プライバシー保護）

---

### UC-2: 管理者が報告を確認しユーザーをBAN

**アクター**: 管理者
**前提条件**: UC-1の報告が作成済み

**フロー**:

1. 管理者が管理ダッシュボードで未処理報告（`status='pending'`）を確認
2. 報告詳細を開く
   - 報告者: ユーザーA
   - 被報告者: ユーザーB
   - 報告理由: harassment
   - 詳細説明: 「何度も不適切な言葉で罵られました」
   - 対象メッセージ: 表示（リンク or 埋め込み）
3. 管理者がメッセージ内容を確認し、BAN判断
4. 管理者が「ユーザーBをBAN」ボタンをクリック
5. BAN理由入力モーダル表示
   - BAN理由: 「ハラスメント行為」
6. BAN実行
   - `users.is_banned = TRUE`（ユーザーB）
   - `users.banned_at = NOW()`
   - `users.ban_reason = 'ハラスメント行為'`
   - `users.banned_by = <管理者ID>`
7. 報告ステータス更新
   - `reports.status = 'resolved'`
   - `reports.resolved_by = <管理者ID>`
   - `reports.resolved_at = NOW()`
   - `reports.admin_notes = 'ユーザーBをBAN実行'`
8. ユーザーBに通知（BAN理由を含む）
9. ユーザーAに通知（報告が対応されたことを通知、詳細は非開示）

**事後条件**:

- ユーザーBはログイン可能だが、依頼投稿・受諾・メッセージ送信が不可
- ユーザーBがアクション実行時に「あなたのアカウントは利用制限されています」エラー表示

---

### UC-3: 管理者が報告を却下

**アクター**: 管理者
**前提条件**: ユーザーが依頼を報告（理由: spam）

**フロー**:

1. 管理者が報告確認
2. 依頼内容を確認し、スパムではないと判断
3. 「報告を却下」ボタンをクリック
4. 却下理由入力
   - 管理者メモ: 「正当な依頼と判断。スパムには該当しない」
5. 報告ステータス更新
   - `reports.status = 'rejected'`
   - `reports.resolved_by = <管理者ID>`
   - `reports.resolved_at = NOW()`
   - `reports.admin_notes = '正当な依頼と判断。スパムには該当しない'`
6. 報告者に通知（「報告を確認しましたが、規約違反には該当しませんでした」）

**事後条件**:

- 被報告者（依頼投稿者）には何も通知されない
- 報告は却下済みとして記録が残る

---

## 実装計画

### Phase 1（今回実装）: 報告機能の基盤整備

#### ステップ1: データベースマイグレーション

**マイグレーションファイル**:

```bash
npx supabase migration new add_reports_table
```

**含まれる変更**:

1. **`reports` テーブル作成**
   - 報告対象: ユーザー、依頼、メッセージ、受け渡し
   - ステータス管理: pending, reviewing, resolved, rejected
   - インデックス作成
   - トリガー設定（updated_at自動更新）

2. **`users` テーブル拡張**（スキーマのみ準備）
   - `role` カラム追加（'user', 'admin'）
   - `is_banned` カラム追加（デフォルトFALSE）
   - `banned_at`, `ban_reason`, `banned_by` カラム追加
   - インデックス作成

3. **RLSポリシー設定**
   - 一般ユーザー: 自分の報告のみ閲覧可能
   - 一般ユーザー: 報告作成可能（BANされていない場合のみ）
   - 管理者権限: Phase 2で有効化（今回はコメント化）

**推定作業時間**: 1時間

#### ステップ2: ユーザー側報告UI実装

**優先順位**: P0（MVPコア機能）

**必要な機能**:

1. **報告ボタン追加**（各エンティティ）
   - ユーザープロフィール → 「このユーザーを報告」
   - 依頼詳細 → 「この依頼を報告」
   - メッセージ → 「このメッセージを報告」（長押しメニュー）
   - 受け渡し詳細 → 「この受け渡しを報告」

2. **報告フォーム**（共通モーダル）
   - 報告理由選択（ドロップダウン）
   - 詳細説明（テキストエリア、最低20文字）
   - キャンセル・送信ボタン

3. **報告完了確認画面**
   - 「報告を受け付けました」メッセージ
   - 「運営チームが確認します」説明

**実装ファイル**:

- `src/components/features/reports/ReportButton.tsx`
- `src/components/features/reports/ReportModal.tsx`
- `src/lib/actions/reports.ts` (Server Action)
- `src/lib/validations/reports.ts` (Zodスキーマ)

**推定作業時間**: 3-4時間

#### ステップ3: 運用準備

**手動運用手順書作成**:

1. **報告確認手順**
   - Supabase StudioでreportsテーブルをSQL照会
   - `SELECT * FROM reports WHERE status = 'pending' ORDER BY created_at DESC;`

2. **BAN実行手順**（緊急時）
   - SQLで手動実行: `UPDATE users SET is_banned = TRUE WHERE id = '<user_id>';`
   - 報告ステータス更新: `UPDATE reports SET status = 'resolved' WHERE id = '<report_id>';`

3. **エスカレーション基準**
   - 同一ユーザーへの報告が3件以上: 要確認
   - 詐欺・ハラスメント報告: 即時確認

**ドキュメント**: `docs/implementation/moderation-manual-operation.md`

**推定作業時間**: 30分

---

### Phase 2（TODO・将来実装）: 管理者UI + BAN機能

**タイミング**: MVP実証実験後、ユーザー数増加時

**必要な機能**:

1. **管理者ダッシュボード**（`/admin`）
   - 未処理報告一覧
   - 統計情報（報告カテゴリ別、ステータス別）
   - BAN済みユーザー数

2. **報告詳細ページ**（`/admin/reports/[id]`）
   - 報告内容表示
   - 報告対象のプレビュー（メッセージ、依頼等）
   - BAN実行ボタン
   - 却下ボタン
   - 管理者メモ入力

3. **BANユーザー管理ページ**（`/admin/banned-users`）
   - BAN済みユーザー一覧
   - BAN解除機能
   - BAN理由編集

4. **BAN機能のRLSポリシー有効化**
   - BANユーザーは投稿・受諾・メッセージ不可
   - 閲覧のみ可能

**推定作業時間**: 6-8時間

---

## Phase 2実装時の検討事項（TODO）

### 🔴 Phase 2で決定が必要な事項

#### 1. レビュー報告機能

- レビュー（reviews）も報告対象に含めるか？
- 例: 「報復レビュー」「虚偽のレビュー」
- 実装コスト: 小（reportsテーブルに`reported_review_id`追加のみ）

#### 2. 自動BAN機能

- 一定数の報告で自動BANするか？
- 例: 同一ユーザーへの報告が5件以上で自動BAN
- メリット: 迅速な対応
- デメリット: 誤BAN（虚偽報告の悪用）
- 推奨: まず手動運用で実績を見てから判断

#### 3. 段階的ペナルティ

- 段階的なペナルティを導入するか？
  - 警告（1回目）
  - 一時停止（2回目、7日間）
  - 永久BAN（3回目）
- スキーマ変更: `users.ban_type` ('warning', 'temporary', 'permanent') 追加
- 実装コスト: 中（UI + ロジック + タイマー機能）

#### 4. 証拠画像の添付

- 報告時に証拠画像をアップロード可能にするか？
- メリット: 管理者の判断が容易
- デメリット: ストレージコスト、実装工数増
- スキーマ変更: `reports.evidence_images` (TEXT[]) 追加

#### 5. BANされたユーザーの既存取引

- BANされたユーザーが進行中の取引を持っている場合の処理は？
  - **オプションA**: 進行中の取引は継続可能（完了まで）← 推奨
  - **オプションB**: 即座に全取引キャンセル
  - **オプションC**: 管理者が個別判断（取引継続 or キャンセル）
- 推奨理由: 依頼者保護（代行者がBANされても買い物は完了させる）

---

### 🟡 Phase 3以降の機能拡張アイデア

- 報告の重複検知（同一エンティティへの複数報告を統合表示）
- 報告者への進捗通知（「報告が確認されました」「対応が完了しました」）
- BANユーザーの異議申し立て機能
- 管理者アクティビティログ（誰が・いつ・何をBANしたか）
- 自動モデレーション（AIによる不適切コンテンツ検知）
- 通報者の完全匿名化オプション

---

## 付録

### 報告理由カテゴリの詳細

| カテゴリ       | 英語名                  | 対象                       | 例                     |
| -------------- | ----------------------- | -------------------------- | ---------------------- |
| 禁止品目       | `prohibited_items`      | 依頼                       | 酒類、医薬品、たばこ   |
| ハラスメント   | `harassment`            | ユーザー、メッセージ       | 誹謗中傷、脅迫         |
| 詐欺           | `fraud`                 | ユーザー、依頼、受け渡し   | 代金未払い、商品未受領 |
| 不適切な内容   | `inappropriate_content` | ユーザー、依頼、メッセージ | 性的・暴力的コンテンツ |
| スパム         | `spam`                  | ユーザー、依頼、メッセージ | 宣伝、繰り返し投稿     |
| 偽プロフィール | `fake_profile`          | ユーザー                   | なりすまし、虚偽情報   |
| 支払いトラブル | `payment_issue`         | 受け渡し                   | 金額相違、支払い拒否   |
| その他         | `other`                 | 全て                       | 上記に該当しない問題   |

### ステータス遷移図

```
pending (未処理)
  ↓
reviewing (確認中) ← 管理者が確認開始
  ↓
  ├─→ resolved (対応完了) ← BAN実行 or 問題対応完了
  └─→ rejected (却下) ← 規約違反なしと判断
```

### SQL実装例: BAN実行関数

```sql
-- BAN実行関数（管理者専用）
CREATE OR REPLACE FUNCTION ban_user(
  target_user_id UUID,
  reason TEXT,
  admin_id UUID
)
RETURNS VOID AS $$
BEGIN
  -- 管理者権限チェック
  IF NOT EXISTS (SELECT 1 FROM users WHERE id = admin_id AND role = 'admin') THEN
    RAISE EXCEPTION 'Unauthorized: Only admins can ban users';
  END IF;

  -- ユーザーをBAN
  UPDATE users
  SET
    is_banned = TRUE,
    banned_at = NOW(),
    ban_reason = reason,
    banned_by = admin_id
  WHERE id = target_user_id;

  -- 通知作成（BANされたユーザーに通知）
  INSERT INTO notifications (user_id, type, title, body)
  VALUES (
    target_user_id,
    'account_banned',
    'アカウントが利用制限されました',
    format('理由: %s', reason)
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

---

## 次のステップ

### Phase 1実装の進め方

#### ✅ 準備完了

- 設計ドキュメント作成完了（このファイル）
- 実装範囲確定（報告機能のみ、BAN機能は設計のみ）

#### 🔄 次のアクション

1. **マイグレーションファイル作成**

   ```bash
   npx supabase migration new add_reports_table
   ```

   - reportsテーブル作成
   - usersテーブル拡張（role, is_banned等）
   - RLSポリシー設定

2. **マイグレーション適用とテスト**

   ```bash
   npx supabase start
   npx supabase db reset
   ```

3. **型定義生成**

   ```bash
   npx supabase gen types typescript --local > src/lib/supabase/types.ts
   ```

4. **報告UI実装**（Phase 3の依頼機能実装後）
   - ReportButton.tsx
   - ReportModal.tsx
   - Server Actions
   - Zodバリデーション

5. **運用手順書作成**
   - `docs/implementation/moderation-manual-operation.md`
   - Supabase Studioでの報告確認手順
   - 緊急時のBAN実行手順

---

**承認待ち**: このドキュメントの設計でOKであれば、マイグレーションファイル作成を開始します。
