# レシート管理方針ドキュメント

**作成日**: 2025-10-12
**目的**: Kotokuプラットフォームにおけるレシートの取り扱い方針（原本 vs デジタル画像）を定義
**対象**: MVP Phase 1 および Phase 2以降の拡張計画

---

## 📋 エグゼクティブサマリー

### 結論

**MVP期（Phase 1）**: レシートの取り扱いは**依頼者が選択**（インフォームド・コンセント型UX）

#### レシート分離 vs 混在

| 項目             | 設定                         | デフォルト        |
| ---------------- | ---------------------------- | ----------------- |
| **レシート分離** | `receipt_separated_required` | ✅ `true`（推奨） |
| **レシート原本** | `receipt_original_required`  | `false`（任意）   |

#### 選択肢

**1. レシート分離 + 原本不要（デフォルト・推奨）**

- レシートを依頼ごとに分けて会計
- デジタル画像のみでOK

**2. レシート分離 + 原本必要**

- レシートを依頼ごとに分けて会計
- レシート原本も受け渡し

**3. レシート混在OK + 原本不要（UX重視）**

- 他の買い物と一緒に会計してもOK
- デジタル画像のみ
- ⚠️ 返品困難、金額不明確、経費計上不可

#### スキーマ変更

```sql
-- requestsテーブルに追加
receipt_separated_required  boolean DEFAULT true   -- レシート分離が必要か
receipt_original_required   boolean DEFAULT false  -- レシート原本が必要か
```

**制約ロジック**: `receipt_original_required = true` なら `receipt_separated_required`も強制的に`true`

**Phase 2以降**: OCR自動入力、混在レシートの自動分離を検討

---

## 1️⃣ 背景と問題提起

### 現在の設計

```sql
CREATE TABLE handovers (
  id uuid PRIMARY KEY,
  request_id uuid UNIQUE NOT NULL,
  receipt_image_url text,  -- レシート画像URL
  total_amount int,
  ...
);
```

- 代行者がレシート画像をアップロード
- 依頼者はアプリで画像を確認できる
- **物理的なレシート原本の取り扱いは未定義**

### 問題

**問題A**: 「レシート原本は依頼者に渡す必要があるのか？」

**問題B**: 「複数依頼を同時処理する場合、レシートは分けるべきか？混在OKか？」

これらには以下の観点からの検討が必要:

1. 法的要件（税務申告、電子帳簿保存法）
2. 返品・交換・保証
3. トラブル防止・紛争解決
4. UXとオペレーション（代行者の負担 vs 依頼者の安心感）

---

## 2️⃣ 法的要件の分析

### 税務申告の観点（依頼者側）

| ケース                                   | レシート原本の必要性 | 理由                                                           |
| ---------------------------------------- | -------------------- | -------------------------------------------------------------- |
| **個人消費**（日常の買い物）             | ❌ 不要              | 経費ではないので税務申告不要                                   |
| **経費計上**（個人事業主・フリーランス） | ⚠️ 原則必要          | 税務申告の証拠書類として必要                                   |
| **電子帳簿保存法対応**                   | ✅ 画像でもOK        | 一定の要件（真実性・可視性）を満たせばデジタル画像も認められる |

#### 電子帳簿保存法（2022年1月改正）

**要件**:

1. **真実性の確保**: タイムスタンプまたは訂正削除履歴の保持
2. **可視性の確保**: 検索機能、ディスプレイ・プリンタの備付け

**Kotokuの対応**:

- レシート画像のメタデータ（撮影日時、GPS）を保存
- アプリで検索・閲覧可能
- → 電子帳簿保存法の要件を満たす可能性が高い

**結論**:

- 経費計上する依頼者は、原則としてレシート原本が必要
- ただし、電子帳簿保存法の要件を満たせば画像でもOK
- → **依頼者に選択肢を提供すべき**

---

## 3️⃣ 返品・交換・保証の観点

### 一般的な小売店のルール

- **返品・交換**: 原則としてレシート原本が必要
- **保証**: 家電製品等の保証にはレシート原本が必要
- **不良品対応**: レシート原本の提示が求められる

### Kotokuでのシナリオ

| シナリオ                           | レシート原本の必要性          |
| ---------------------------------- | ----------------------------- |
| **不良品**（賞味期限切れ、破損等） | ✅ 必要（店舗への返品に必要） |
| **代替品が気に入らない**           | ✅ 必要（返品・交換に必要）   |
| **家電製品の保証**                 | ✅ 必要（初期不良対応に必要） |

### Kotokuの想定ユースケース

- 食料品・日用品がメイン
- 高額商品は禁止（5,000円以上）
- 返品・交換のケースは稀

**結論**: 返品・交換の可能性は低いが、ゼロではない
→ **依頼者が必要な場合は原本を渡せるようにすべき**

---

## 4️⃣ トラブル防止・紛争解決の観点

### シナリオA: 金額の不一致トラブル

```
依頼者: 「レシートには1,200円と書いてあるのに、1,500円請求された」
代行者: 「間違いありません、1,500円です」
```

**対処**: レシート画像がアプリに残っている → **画像で十分**

### シナリオB: レシート偽造・改ざんの疑い

```
依頼者: 「このレシート画像、何か変じゃない？」
```

**リスク評価**:

- 偽造のリスクは低い（謝礼500円のためにリスクを取るメリットがない）
- 画像のメタデータ（撮影日時、GPS）で検証可能
- 原本があればより確実だが、MVP期では過剰

**結論**: 画像で十分（原本は任意）

### シナリオC: 税務調査での証拠提示

```
税務署: 「この経費のレシート原本を見せてください」
依頼者: 「代行してもらった買い物なので、原本は代行者が持っています」
```

**リスク評価**:

- 経費計上する依頼者は少数派
- ただし、フリーランス等が経費計上するケースは想定すべき

**結論**: 経費計上する依頼者のために、**レシート原本を渡すオプション**が必要

---

## 5️⃣ UXとオペレーションの比較

### パターンA: レシート原本を必ず渡す（強制）

**フロー**:

1. 代行者が買い物
2. レシート画像をアップロード
3. **商品とレシート原本を一緒に受け渡し**

| メリット                  | デメリット                              |
| ------------------------- | --------------------------------------- |
| ✅ 依頼者が経費計上できる | ⚠️ 代行者がレシート原本を紛失するリスク |
| ✅ 返品・交換が容易       | ⚠️ 受け渡し時に「渡し忘れた」トラブル   |
| ✅ 透明性が高い           | ⚠️ 紙のゴミが増える（環境負荷）         |

### パターンB: レシート原本は任意（依頼者が選択）✅ **推奨**

**フロー**:

1. **依頼投稿時に「レシート原本が必要ですか？」を選択**
   - はい → 代行者がレシート原本を保管し、受け渡し時に渡す
   - いいえ → レシート画像のみでOK、原本は代行者が廃棄
2. 代行者が買い物
3. レシート画像をアップロード
4. 受け渡し時に「レシート原本が必要」な場合のみ渡す

| メリット                                  | デメリット                                                          |
| ----------------------------------------- | ------------------------------------------------------------------- |
| ✅ 柔軟な対応（依頼者のニーズに合わせる） | ⚠️ 依頼者が後から「やっぱり原本が欲しい」と言った場合に対応できない |
| ✅ 紙のゴミを減らせる                     |                                                                     |
| ✅ 代行者の紛失リスクを減らせる           |                                                                     |

### パターンC: レシート原本は原則不要（デジタル画像のみ）

**フロー**:

1. 代行者が買い物
2. レシート画像をアップロード
3. レシート原本は代行者が保管（一定期間後に廃棄）
4. 依頼者が「原本が必要」と後から言った場合、代行者に連絡して郵送

| メリット                    | デメリット                                    |
| --------------------------- | --------------------------------------------- |
| ✅ シンプルな運用           | ⚠️ 代行者がレシート原本を保管する期間が不明確 |
| ✅ 必要な場合のみ原本を渡す | ⚠️ 郵送の手間とコスト                         |

---

## 5️⃣-2 レシート分離 vs 混在の比較

### ケース1: レシート分離（`receipt_separated_required = true`）✅ **推奨**

**シナリオ**: 複数依頼や自分の買い物と一緒に買う場合でも、**依頼ごとに分けて会計**

```
依頼A: 牛乳、パン → レジ通す（レシートA）
依頼B: 卵、豆腐 → レジ通す（レシートB）
自分の買い物: ビール、お菓子 → レジ通す（レシートC）
```

| メリット                                 | デメリット                                  |
| ---------------------------------------- | ------------------------------------------- |
| ✅ 金額が明確（レシート合計 = 依頼合計） | ❌ 代行者の負担が大きい（複数回レジを通す） |
| ✅ 依頼者が安心（トラブル少ない）        | ❌ 依頼承諾率が下がる可能性                 |
| ✅ 返品・交換が容易                      | ❌ レジが混雑している場合に時間がかかる     |
| ✅ OCR実装が容易（Phase 2）              |                                             |
| ✅ 経費計上可能（原本渡す場合）          |                                             |

**推奨ケース**:

- レシート原本が必要な依頼（`receipt_original_required = true`の場合は強制）
- 返品・交換の可能性がある商品
- 依頼者が金額を明確に確認したい場合

---

### ケース2: レシート混在OK（`receipt_separated_required = false`）⚠️

**シナリオ**: 複数依頼や自分の買い物と一緒に1回で会計してOK

```
全部まとめて1回レジ通す:
- 依頼Aの商品: 牛乳198円、パン148円
- 依頼Bの商品: 卵238円、豆腐98円
- 自分の買い物: ビール350円、お菓子200円
----
レシート合計: 1,232円（1枚のレシートに全部含まれる）
```

| メリット                       | デメリット                                            |
| ------------------------------ | ----------------------------------------------------- |
| ✅ 代行者が楽（1回で会計可能） | ❌ 金額入力が面倒（レシートから該当商品を探して計算） |
| ✅ 依頼承諾率UP                | ❌ 返品・交換が困難（レシート原本がない）             |
| ✅ レジが早い                  | ❌ トラブルリスク高（依頼者が不信感を持ちやすい）     |
|                                | ❌ 商品別の実際価格が不明（`actual_price`入力困難）   |
|                                | ❌ OCR実装困難（Phase 2）                             |
|                                | ❌ 経費計上不可（混在レシートの原本では無意味）       |

**推奨ケース**:

- 日常消費の買い物（経費計上不要）
- 信頼できる代行者
- 返品・交換の可能性が低い商品

**注意事項**:

- 代行者は依頼ごとの合計金額を**正確に計算して入力**する必要がある
- 依頼者は「レシート合計 ≠ 依頼合計」を理解して受け入れる必要がある

---

### 制約ロジック

```typescript
// バリデーションロジック
if (receipt_original_required === true) {
  // レシート原本が必要な場合は、分離も強制
  receipt_separated_required = true

  // 理由: 混在レシートの原本を渡しても経費計上できない
}
```

---

### MVP期の方針

**デフォルト**: `receipt_separated_required = true`（レシート分離・推奨）

**変更可能**: 依頼投稿時に依頼者が「レシート混在OK」を選択可能

**UI設計**: 両方のメリット・デメリットを明示して、インフォームド・コンセント型で選択

---

## 6️⃣ 類似サービスの事例

| サービス                  | レシート原本の扱い | 理由                                       |
| ------------------------- | ------------------ | ------------------------------------------ |
| **TaskRabbit / 家事代行** | ✅ 渡す            | 依頼者が商品の所有権を持つため             |
| **Uber Eats / 出前館**    | ❌ 渡さない        | 配達員が複数注文を同時配達するため管理困難 |
| **Amazon**                | ✅ 渡す（同梱）    | 返品・交換に必要なため                     |
| **メルカリShops**         | ❌ 不要            | 取引履歴がアプリに残るため                 |

**結論**: 買い物代行サービスでは「レシート原本を渡す」が一般的
ただし、デジタル画像での運用も増えている（環境配慮、利便性）

---

## 7️⃣ 最終的な方針決定

### 🎯 推奨方針: **パターンB（任意 - 依頼者が選択）**

#### 理由

1. **法的要件**: 経費計上する依頼者には原本が必要、個人消費には不要
2. **返品・交換**: 不良品の場合に原本が必要、ただし稀
3. **UX**: デフォルトは「原本不要」で紙のゴミを減らす、必要な依頼者は選択可能
4. **実装コスト**: 最小限のスキーマ変更で実装可能

---

## 8️⃣ 実装方針（MVP Phase 1）

### データベーススキーマ変更

```sql
-- requestsテーブルに追加
ALTER TABLE requests
ADD COLUMN receipt_separated_required boolean DEFAULT true,
ADD COLUMN receipt_original_required boolean DEFAULT false;

COMMENT ON COLUMN requests.receipt_separated_required IS
'レシートを依頼ごとに分けて会計する必要があるか（依頼者が選択）。
trueの場合: 他の買い物と一緒に会計不可、依頼ごとにレシート分離
falseの場合: 他の買い物と一緒に会計OK、代行者が合計金額を手動計算';

COMMENT ON COLUMN requests.receipt_original_required IS
'レシート原本が必要かどうか（依頼者が選択）。
経費計上や返品の可能性がある場合はtrue。
trueの場合はreceipt_separated_requiredも強制的にtrueになる。';

-- handoversテーブルに追加
ALTER TABLE handovers
ADD COLUMN receipt_original_delivered boolean DEFAULT false;

COMMENT ON COLUMN handovers.receipt_original_delivered IS
'レシート原本を依頼者に渡したかどうか（代行者が確認）。';

-- 制約チェック（アプリケーション側で実装）
-- receipt_original_required = true の場合、receipt_separated_required も true
```

### UI実装

#### 依頼投稿画面（依頼者）

```typescript
// src/app/(main)/requests/new/page.tsx

<Form>
  {/* 既存のフィールド */}

  <section className="space-y-4">
    <h3 className="font-semibold">レシート取り扱い方法</h3>

    {/* レシート分離 vs 混在の選択 */}
    <RadioGroup name="receipt_policy" defaultValue="separated">
      {/* 選択肢1: レシート分離（推奨） */}
      <Radio value="separated">
        <div className="flex items-start gap-2">
          <input type="radio" />
          <div>
            <strong>レシートを分けてください（推奨）</strong>

            <div className="mt-2 grid grid-cols-2 gap-4 text-sm">
              <div className="space-y-1">
                <p className="font-medium text-green-700">メリット</p>
                <ul className="list-disc list-inside space-y-1 text-gray-600">
                  <li>返品・交換が容易</li>
                  <li>金額が明確で安心</li>
                  <li>トラブルが少ない</li>
                </ul>
              </div>
              <div className="space-y-1">
                <p className="font-medium text-amber-700">デメリット</p>
                <ul className="list-disc list-inside space-y-1 text-gray-600">
                  <li>代行者の負担が増える</li>
                  <li>承諾されにくい可能性</li>
                </ul>
              </div>
            </div>

            {/* レシート分離を選んだ場合の追加オプション */}
            <div className="mt-3 pl-4 border-l-2 border-gray-300">
              <Checkbox name="receipt_original_required">
                レシート原本も必要（経費計上や返品予定の場合）
              </Checkbox>
            </div>
          </div>
        </div>
      </Radio>

      {/* 選択肢2: レシート混在OK */}
      <Radio value="combined">
        <div className="flex items-start gap-2">
          <input type="radio" />
          <div>
            <strong>レシートを分けなくてもOK</strong>

            <div className="mt-2 grid grid-cols-2 gap-4 text-sm">
              <div className="space-y-1">
                <p className="font-medium text-green-700">メリット</p>
                <ul className="list-disc list-inside space-y-1 text-gray-600">
                  <li>代行者が楽（承諾率UP）</li>
                  <li>早く買ってもらえる可能性</li>
                </ul>
              </div>
              <div className="space-y-1">
                <p className="font-medium text-amber-700">デメリット</p>
                <ul className="list-disc list-inside space-y-1 text-gray-600">
                  <li>返品・交換が困難</li>
                  <li>商品別の正確な金額が不明</li>
                  <li>経費計上には使えない</li>
                  <li>トラブル時の説明が難しい</li>
                </ul>
              </div>
            </div>

            <Alert variant="warning" className="mt-3">
              ⚠️ 他の買い物と一緒に会計される可能性があります。
              信頼できる代行者を選んでください。
            </Alert>
          </div>
        </div>
      </Radio>
    </RadioGroup>
  </section>
</Form>
```

#### 受け渡し画面（代行者）

```typescript
// src/app/(main)/requests/[id]/handover/page.tsx

<Form>
  {/* レシート取り扱い方法の表示 */}
  <section>
    <h3>レシート取り扱い</h3>

    {request.receipt_separated_required ? (
      <Alert variant="info">
        📄 <strong>レシートを分けてください</strong>

        <details>
          <summary className="cursor-pointer text-sm mt-2">詳細を見る</summary>

          <div className="mt-2 grid grid-cols-2 gap-4 text-sm">
            <div>
              <p className="font-medium text-green-700">メリット</p>
              <ul className="list-disc list-inside text-gray-600">
                <li>依頼者が安心する</li>
                <li>返品・交換が容易</li>
              </ul>
            </div>
            <div>
              <p className="font-medium text-amber-700">デメリット</p>
              <ul className="list-disc list-inside text-gray-600">
                <li>レジを複数回通す必要がある</li>
                <li>自分の買い物とも分ける必要がある</li>
              </ul>
            </div>
          </div>

          {request.receipt_original_required && (
            <Alert variant="warning" className="mt-2">
              📋 <strong>レシート原本も必要です</strong>
              商品と一緒にレシート原本を渡してください。
            </Alert>
          )}
        </details>
      </Alert>
    ) : (
      <Alert variant="success">
        ✅ <strong>レシートを分けなくてOK</strong>

        <details>
          <summary className="cursor-pointer text-sm mt-2">詳細を見る</summary>

          <div className="mt-2 grid grid-cols-2 gap-4 text-sm">
            <div>
              <p className="font-medium text-green-700">メリット</p>
              <ul className="list-disc list-inside text-gray-600">
                <li>自分の買い物や他の依頼と一緒に会計できる</li>
                <li>レジを1回で済む</li>
              </ul>
            </div>
            <div>
              <p className="font-medium text-amber-700">注意点</p>
              <ul className="list-disc list-inside text-gray-600">
                <li>商品別の正確な金額を記録する必要がある</li>
                <li>返品・交換ができない可能性</li>
              </ul>
            </div>
          </div>

          <HelpText className="mt-2">
            レシートに他の商品が含まれている場合は、
            受け渡し時に「この依頼の商品」の合計金額を正確に入力してください。
          </HelpText>
        </details>
      </Alert>
    )}
  </section>

  {/* レシート画像アップロード */}
  <FileUpload name="receipt_image" required />

  {/* レシート混在の場合の確認 */}
  {!request.receipt_separated_required && (
    <Alert variant="info">
      ℹ️ このレシートには他の買い物も含まれていますか？

      <RadioGroup name="receipt_type" className="mt-2">
        <Radio value="separated">
          いいえ、この依頼のみです
          → レシート合計金額をそのまま入力してください
        </Radio>

        <Radio value="combined">
          はい、他の商品も含まれています
          → この依頼の商品の合計金額を手動で計算して入力してください

          <HelpText className="ml-6 mt-1">
            例: レシート合計1,232円のうち、
            この依頼分は346円（牛乳198円＋パン148円）
          </HelpText>
        </Radio>
      </RadioGroup>
    </Alert>
  )}

  {/* 合計金額入力 */}
  <FormField label="合計金額（円）" required>
    <Input
      type="number"
      name="total_amount"
      placeholder={
        request.receipt_separated_required
          ? "レシート合計金額"
          : "この依頼の商品の合計金額"
      }
    />
  </FormField>

  {/* レシート原本の確認 */}
  {request.receipt_original_required && (
    <Checkbox name="receipt_original_delivered" required>
      レシート原本を依頼者に渡しました
    </Checkbox>
  )}
</Form>
```

#### 依頼詳細画面（依頼者）

```typescript
// src/app/(main)/requests/[id]/page.tsx

<HandoverSection>
  <h2>受け渡し情報</h2>

  <Image src={handover.receipt_image_url} alt="レシート画像" />

  <p>合計金額: {handover.total_amount}円</p>

  {request.receipt_original_required && (
    <StatusBadge variant={handover.receipt_original_delivered ? "success" : "warning"}>
      {handover.receipt_original_delivered
        ? "✅ レシート原本を受け取りました"
        : "⏳ レシート原本は受け渡し時に渡されます"}
    </StatusBadge>
  )}

  <Button onClick={() => confirmHandover()}>
    商品を受け取りました
  </Button>
</HandoverSection>
```

### Validation（Zod）

```typescript
// lib/validations/request.ts
import { z } from 'zod'

export const createRequestSchema = z
  .object({
    title: z.string().min(1).max(100),
    description: z.string().max(500).optional(),
    reward_amount: z.number().int().positive().default(500),
    receipt_separated_required: z.boolean().default(true), // 新規追加
    receipt_original_required: z.boolean().default(false), // 新規追加
    items: z
      .array(
        z.object({
          item_name: z.string().min(1),
          quantity: z.number().int().positive().default(1),
          price_cap: z.number().int().positive().optional(),
          substitute_allowed: z.boolean().default(false),
        })
      )
      .min(1),
  })
  .refine(
    (data) => {
      // 制約: レシート原本が必要な場合は、レシート分離も必須
      if (data.receipt_original_required && !data.receipt_separated_required) {
        return false
      }
      return true
    },
    {
      message: 'レシート原本が必要な場合は、レシート分離も必須です',
      path: ['receipt_separated_required'],
    }
  )

// lib/validations/handover.ts
export const createHandoverSchema = z.object({
  receipt_image_url: z.string().url(),
  total_amount: z.number().int().positive(),
  receipt_original_delivered: z.boolean().default(false), // 新規追加
  product_images: z.array(z.string().url()).optional(),
  is_sealed: z.boolean().default(true),
})
```

---

## 9️⃣ ガイドライン

### 依頼者向けガイドライン

#### レシート取り扱い方法の選択

**「レシートを分けてください」を選ぶべきケース**（デフォルト・推奨）:

- ✅ 金額を明確に確認したい場合
- ✅ 返品・交換の可能性がある商品
- ✅ 経費計上する場合（レシート原本も必要）
- ✅ 初めて依頼する代行者の場合

**「レシートを分けなくてもOK」を選ぶケース**:

- ✅ 信頼できる代行者を知っている場合
- ✅ 早く買ってほしい場合（承諾率UP）
- ✅ 日常消費の買い物（経費計上不要）
- ⚠️ 返品・交換のリスクを理解している場合

**「レシート原本が必要」を選ぶべきケース**:

- ✅ 個人事業主やフリーランスで、経費計上する場合
- ✅ 返品・交換の可能性が高い商品
- ✅ 高額商品（家電等）で保証が必要な場合
- ⚠️ この場合、「レシートを分けてください」も強制的に選択されます

---

### 代行者向けガイドライン

#### レシート分離が必要な場合（`receipt_separated_required = true`）

**手順**:

1. 複数依頼や自分の買い物がある場合でも、**依頼ごとに分けて会計**
2. レジを複数回通す必要がある（手間はかかるが、トラブル防止のため重要）
3. レシート画像をアップロード
4. レシート合計金額をそのまま入力

**注意点**:

- 自分の買い物とも必ず分ける
- レシート原本が必要な場合は、紛失しないように保管

---

#### レシート混在OKの場合（`receipt_separated_required = false`）

**手順**:

1. 他の買い物と一緒に会計してOK
2. レシート画像をアップロード（他の商品も含まれている可能性あり）
3. **この依頼の商品の合計金額を手動で計算**して入力
   - 例: レシート合計1,232円のうち、牛乳198円＋パン148円 = 346円

**注意点**:

- 金額計算を間違えないこと（トラブルの元）
- 可能であれば商品別の価格も記録する（`actual_price`フィールド）
- 返品・交換はできない可能性があることを認識

---

#### レシート原本が必要な場合

**手順**:

1. レシート画像をアップロード
2. レシート原本を**紛失しないように保管**
3. 受け渡し時に商品と一緒にレシート原本を渡す
4. アプリで「レシート原本を渡しました」にチェック

**レシート原本が不要な場合**:

1. レシート画像をアップロード
2. レシート原本は廃棄してOK（環境にも優しい）

---

## 🔟 Phase 2以降の拡張計画

### Phase 2A: 依頼者が後から原本を希望した場合の対応

**シナリオ**:

```
依頼者: 「やっぱり経費計上したいので、レシート原本が欲しいです」
→ 依頼投稿時は「原本不要」を選択していた
```

**対応案**:

1. 代行者にチャットで連絡
2. 代行者がまだレシート原本を保管している場合:
   - 郵送（送料は依頼者負担？代行者負担？運営負担？）
   - または次回の受け渡し時に渡す
3. 代行者がすでに廃棄した場合:
   - レシート画像のみで対応（電子帳簿保存法の要件を満たす）

**実装優先度**: 🟡 中（MVP期のユーザーフィードバック次第）

### Phase 2B: レシート原本の保管期間ルール

**課題**:

- 代行者がレシート原本を保管する期間は？
- 一定期間後に廃棄してOK？

**提案**:

- **保管期間: 受け渡し完了後30日間**
- 30日以内に依頼者が「原本が欲しい」と言った場合は郵送
- 30日経過後は廃棄OK

**実装優先度**: 🟡 中（MVP期のユーザーフィードバック次第）

---

## 1️⃣1️⃣ リスク管理

### 🟢 低リスク事項

- ✅ 法的要件: クリア（依頼者が選択できる）
- ✅ UX: デフォルトが「原本不要」なので紙のゴミは最小限
- ✅ 実装コスト: 最小限のスキーマ変更で実装可能

### 🟡 中リスク事項

- ⚠️ 依頼者が後から「原本が欲しい」と言った場合の対応
  - **対策**: Phase 2で郵送対応を検討
  - **モニタリング**: MVP期のユーザーフィードバック収集

- ⚠️ 代行者がレシート原本を紛失した場合
  - **対策**: ガイドラインで「紛失しないように保管」と明示
  - **補償**: 依頼者が経費計上できなくなった場合の補償ルール（Phase 2で検討）

### 🔴 高リスク事項（回避済み）

- なし（依頼者が選択できるので柔軟に対応可能）

---

## 1️⃣2️⃣ 実装タスク（MVP Phase 1）

### ✅ 必須タスク

1. **データベーススキーマ変更**
   - `requests.receipt_separated_required`を追加（DEFAULT true）
   - `requests.receipt_original_required`を追加（DEFAULT false）
   - `handovers.receipt_original_delivered`を追加（DEFAULT false）
   - マイグレーションファイル作成

2. **UI実装**
   - 依頼投稿画面:
     - レシート分離 vs 混在の選択（ラジオボタン）
     - Pros/Cons表示（メリット・デメリット）
     - レシート原本チェックボックス（分離選択時のみ）
   - 受け渡し画面（代行者）:
     - レシート取り扱い方法の表示（Pros/Cons）
     - レシート混在時の確認UI（他の商品が含まれているか）
     - 合計金額入力（分離 or 混在で説明が変わる）
   - 依頼詳細画面（依頼者）:
     - レシート取り扱い方法の表示
     - レシート原本の受け渡しステータス表示

3. **Validation実装**
   - `createRequestSchema`に`receipt_separated_required`と`receipt_original_required`追加
   - 制約ロジック: `receipt_original_required = true`なら`receipt_separated_required`も強制的に`true`
   - `createHandoverSchema`に`receipt_original_delivered`追加

4. **ガイドライン追加**
   - FAQ: 「レシート分離 vs 混在はどう選ぶ？」
   - FAQ: 「レシート原本は必要ですか？」
   - ヘルパー向けガイドライン: レシート分離の方法
   - ヘルパー向けガイドライン: レシート混在時の金額計算方法

### 🔄 Phase 2以降のタスク

1. **依頼者が後から原本を希望した場合の対応**
   - チャット機能での連絡
   - 郵送手続きのフロー

2. **レシート原本の保管期間ルール**
   - 保管期間を30日と定義
   - 代行者向けガイドラインに明記

---

## 1️⃣3️⃣ 成功指標（KPI）

### MVP期（Phase 1）

| 指標                                   | 目標  | 測定方法                                          |
| -------------------------------------- | ----- | ------------------------------------------------- |
| **レシート原本が必要な依頼の割合**     | < 20% | `requests.receipt_original_required = true`の割合 |
| **レシート原本の渡し忘れトラブル**     | < 5%  | サポート問い合わせ件数                            |
| **依頼者が後から原本を希望するケース** | < 10% | チャットでの問い合わせ件数                        |

### Phase 2以降

| 指標                           | 目標 | 測定方法               |
| ------------------------------ | ---- | ---------------------- |
| **郵送対応の発生率**           | < 5% | 郵送リクエスト件数     |
| **レシート原本の紛失トラブル** | < 1% | サポート問い合わせ件数 |

---

## 1️⃣4️⃣ 参考資料

### 法律・規制

- [電子帳簿保存法（e-Tax）](https://www.nta.go.jp/law/joho-zeikaishaku/sonota/jirei/02.htm)
- [国税庁: 電子帳簿保存法一問一答](https://www.nta.go.jp/law/joho-zeikaishaku/sonota/jirei/02.htm)

### 類似サービス

- [TaskRabbit ヘルプセンター](https://support.taskrabbit.com/)
- [Uber Eats パートナーガイド](https://merchants.ubereats.com/)

---

## 1️⃣5️⃣ 変更履歴

| 日付       | バージョン | 変更内容 | 作成者      |
| ---------- | ---------- | -------- | ----------- |
| 2025-10-12 | 1.0.0      | 初版作成 | Claude Code |

---

**次のステップ**: データベーススキーマ変更のマイグレーションファイル作成

**関連ドキュメント**:

- `docs/data-model.md` - データベース設計
- `docs/mvp-features.md` - MVP機能仕様
- `/tmp/kotoku-db-schema-review.md` - スキーマレビュー（レシート管理UX分析結果を含む）
