# 🔌 API仕様書

## プロジェクト名：showya

---

## 1. 🌐 API概要

### ベースURL
- **開発環境**: `http://localhost:3000/api/v1`
- **ステージング**: `https://staging.showya.app/api/v1`
- **本番環境**: `https://showya.app/api/v1`

### APIバージョニング戦略

#### バージョン管理方針
- **URLパス方式**: `/api/v1/`, `/api/v2/` を使用
- **後方互換性**: 最低2つのメジャーバージョンをサポート
- **非推奨化プロセス**: 6ヶ月前に事前通知、段階的廃止

#### バージョン履歴
```
v1.0 (現在): 初期リリース
- 基本的なCRUD操作
- 認証・認可
- ファイルアップロード

v1.1 (予定): 機能拡張
- Bulk操作
- 高度な検索
- Webhook

v2.0 (将来): 大幅な変更
- GraphQL対応
- リアルタイム機能
- AI機能強化
```

#### バージョン指定方法
```http
# URLパス（推奨）
GET /api/v1/profiles

# Acceptヘッダー（代替）
GET /api/profiles
Accept: application/vnd.showya.v1+json

# カスタムヘッダー（代替）
GET /api/profiles
API-Version: v1
```

### 認証方式
- **JWT Bearer Token** (Supabase Auth)
- **Header**: `Authorization: Bearer <token>`

### レスポンス形式
```typescript
// 成功レスポンス
interface SuccessResponse<T> {
  success: true;
  data: T;
  message?: string;
}

// エラーレスポンス
interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: any;
  };
}
```

### HTTPステータスコード
- `200` - 成功
- `201` - 作成成功
- `400` - リクエストエラー
- `401` - 認証エラー
- `403` - 権限エラー
- `404` - リソースが見つからない
- `409` - 競合エラー
- `422` - バリデーションエラー
- `429` - レート制限
- `500` - サーバーエラー

---

## 2. 🔐 認証API

### POST /api/auth/signup
**新規ユーザー登録**

```typescript
// リクエスト
interface SignupRequest {
  email: string;
  password: string;
  confirmPassword: string;
}

// レスポンス
interface SignupResponse {
  user: {
    id: string;
    email: string;
    emailVerified: boolean;
  };
  session: {
    accessToken: string;
    refreshToken: string;
    expiresAt: number;
  };
}
```

**例**
```bash
curl -X POST /api/auth/signup \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "securePassword123",
    "confirmPassword": "securePassword123"
  }'
```

### POST /api/auth/signin
**ログイン**

```typescript
// リクエスト
interface SigninRequest {
  email: string;
  password: string;
  rememberMe?: boolean;
}

// レスポンス
interface SigninResponse {
  user: {
    id: string;
    email: string;
    profile?: {
      username: string;
      displayName: string;
    };
  };
  session: {
    accessToken: string;
    refreshToken: string;
    expiresAt: number;
  };
}
```

### POST /api/auth/signout
**ログアウト**

```typescript
// リクエスト
// Body不要（Authorizationヘッダーのみ）

// レスポンス
interface SignoutResponse {
  message: string;
}
```

### POST /api/auth/reset-password
**パスワードリセット**

```typescript
// リクエスト
interface ResetPasswordRequest {
  email: string;
}

// レスポンス
interface ResetPasswordResponse {
  message: string;
}
```

---

## 3. 👤 プロフィールAPI

### GET /api/profiles/[username]
**公開プロフィール取得**

```typescript
// レスポンス
interface PublicProfileResponse {
  profile: {
    username: string;
    displayName: string;
    title?: string;
    bio?: string;
    avatarUrl?: string;
    seoTitle?: string;
    seoDescription?: string;
    socialLinks: SocialLink[];
    portfolios: Portfolio[];
  };
}

interface SocialLink {
  id: string;
  platform: 'twitter' | 'instagram' | 'youtube' | 'github' | 'custom';
  url: string;
  title?: string;
  iconUrl?: string;
  displayOrder: number;
}

interface Portfolio {
  id: string;
  title: string;
  subtitle?: string;
  description?: string;
  type: 'image' | 'video' | 'website';
  fileUrl?: string;
  websiteUrl?: string;
  thumbnailUrl?: string;
  displayOrder: number;
  isFeatured: boolean;
  tags: string[];
}
```

### GET /api/profiles/me
**自分のプロフィール取得**

```typescript
// レスポンス
interface MyProfileResponse {
  profile: {
    id: string;
    username: string;
    displayName: string;
    title?: string;
    bio?: string;
    avatarUrl?: string;
    customDomain?: string;
    seoTitle?: string;
    seoDescription?: string;
    isPublic: boolean;
    createdAt: string;
    updatedAt: string;
  };
}
```

### PUT /api/profiles/me
**プロフィール更新**

```typescript
// リクエスト
interface UpdateProfileRequest {
  displayName?: string;
  title?: string;
  bio?: string;
  avatarUrl?: string;
  seoTitle?: string;
  seoDescription?: string;
  isPublic?: boolean;
}

// レスポンス
interface UpdateProfileResponse {
  profile: MyProfileResponse['profile'];
}
```

### POST /api/profiles/check-username
**ユーザー名重複チェック**

```typescript
// リクエスト
interface CheckUsernameRequest {
  username: string;
}

// レスポンス
interface CheckUsernameResponse {
  available: boolean;
  suggestions?: string[];
}
```

---

## 4. 🔗 ソーシャルリンクAPI

### GET /api/social-links
**リンク一覧取得**

```typescript
// レスポンス
interface SocialLinksResponse {
  links: SocialLinkWithStats[];
}

interface SocialLinkWithStats extends SocialLink {
  clickCount: number;
  lastClickedAt?: string;
  monthlyClicks: number;
}
```

### POST /api/social-links
**リンク追加**

```typescript
// リクエスト
interface CreateSocialLinkRequest {
  platform: 'twitter' | 'instagram' | 'youtube' | 'github' | 'custom';
  url: string;
  title?: string; // カスタムリンクの場合必須
  iconUrl?: string;
}

// レスポンス
interface CreateSocialLinkResponse {
  link: SocialLinkWithStats;
}
```

### PUT /api/social-links/[id]
**リンク更新**

```typescript
// リクエスト
interface UpdateSocialLinkRequest {
  url?: string;
  title?: string;
  iconUrl?: string;
  displayOrder?: number;
}

// レスポンス
interface UpdateSocialLinkResponse {
  link: SocialLinkWithStats;
}
```

### DELETE /api/social-links/[id]
**リンク削除**

```typescript
// レスポンス
interface DeleteSocialLinkResponse {
  message: string;
}
```

### POST /api/social-links/[id]/click
**クリック数カウント**

```typescript
// リクエスト
interface ClickSocialLinkRequest {
  referrer?: string;
  userAgent?: string;
}

// レスポンス
interface ClickSocialLinkResponse {
  clickCount: number;
}
```

### PUT /api/social-links/reorder
**リンク並び替え**

```typescript
// リクエスト
interface ReorderSocialLinksRequest {
  linkIds: string[]; // 新しい順序でのID配列
}

// レスポンス
interface ReorderSocialLinksResponse {
  links: SocialLinkWithStats[];
}
```

---

## 5. 🎨 ポートフォリオAPI

### GET /api/portfolios
**ポートフォリオ一覧取得**

```typescript
// クエリパラメータ
interface PortfoliosQuery {
  page?: number;
  limit?: number;
  type?: 'image' | 'video' | 'website';
  featured?: boolean;
  tags?: string; // カンマ区切り
}

// レスポンス
interface PortfoliosResponse {
  portfolios: Portfolio[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}
```

### POST /api/portfolios
**ポートフォリオ追加**

```typescript
// リクエスト
interface CreatePortfolioRequest {
  title: string;
  subtitle?: string;
  description?: string;
  type: 'image' | 'video' | 'website';
  fileUrl?: string; // 画像・動画の場合
  websiteUrl?: string; // ウェブサイトの場合
  tags?: string[];
  isFeatured?: boolean;
}

// レスポンス
interface CreatePortfolioResponse {
  portfolio: Portfolio;
}
```

### PUT /api/portfolios/[id]
**ポートフォリオ更新**

```typescript
// リクエスト
interface UpdatePortfolioRequest {
  title?: string;
  subtitle?: string;
  description?: string;
  websiteUrl?: string;
  tags?: string[];
  isFeatured?: boolean;
  displayOrder?: number;
}

// レスポンス
interface UpdatePortfolioResponse {
  portfolio: Portfolio;
}
```

### DELETE /api/portfolios/[id]
**ポートフォリオ削除**

```typescript
// レスポンス
interface DeletePortfolioResponse {
  message: string;
}
```

### POST /api/portfolios/upload
**ファイルアップロード**

```typescript
// リクエスト（multipart/form-data）
interface UploadFileRequest {
  file: File; // 画像または動画ファイル
  type: 'image' | 'video';
}

// レスポンス
interface UploadFileResponse {
  fileUrl: string;
  thumbnailUrl?: string; // 動画の場合
  metadata: {
    filename: string;
    size: number;
    mimeType: string;
    width?: number;
    height?: number;
    duration?: number; // 動画の場合
  };
}
```

### PUT /api/portfolios/reorder
**ポートフォリオ並び替え**

```typescript
// リクエスト
interface ReorderPortfoliosRequest {
  portfolioIds: string[];
}

// レスポンス
interface ReorderPortfoliosResponse {
  portfolios: Portfolio[];
}
```

---

## 6. 🔍 SEO API

### GET /api/seo/generate
**SEOタグ自動生成**

```typescript
// レスポンス
interface GenerateSEOResponse {
  seo: {
    title: string;
    description: string;
    keywords: string[];
    ogTitle: string;
    ogDescription: string;
    ogImage?: string;
  };
}
```

### PUT /api/seo/update
**SEOタグ手動更新**

```typescript
// リクエスト
interface UpdateSEORequest {
  title?: string;
  description?: string;
  keywords?: string[];
  ogTitle?: string;
  ogDescription?: string;
  ogImage?: string;
}

// レスポンス
interface UpdateSEOResponse {
  seo: {
    title: string;
    description: string;
    keywords: string[];
    ogTitle: string;
    ogDescription: string;
    ogImage?: string;
  };
}
```

---

## 7. 📊 分析API

### GET /api/analytics/overview
**概要統計取得**

```typescript
// クエリパラメータ
interface AnalyticsQuery {
  period?: '7d' | '30d' | '90d' | '1y';
  timezone?: string;
}

// レスポンス
interface AnalyticsOverviewResponse {
  stats: {
    totalClicks: number;
    totalViews: number;
    totalLinks: number;
    totalPortfolios: number;
    clicksChange: number; // 前期比較（%）
    viewsChange: number;
  };
  recentActivity: {
    type: 'click' | 'view' | 'update';
    description: string;
    timestamp: string;
  }[];
}
```

### GET /api/analytics/clicks
**クリック分析**

```typescript
// レスポンス
interface ClickAnalyticsResponse {
  clicksByLink: {
    linkId: string;
    platform: string;
    title: string;
    clicks: number;
    clicksChange: number;
  }[];
  clicksByDate: {
    date: string;
    clicks: number;
  }[];
}
```

---

## 8. 💳 サブスクリプションAPI

### GET /api/subscription
**サブスクリプション状況取得**

```typescript
// レスポンス
interface SubscriptionResponse {
  subscription: {
    planType: 'free' | 'pro';
    status: 'active' | 'canceled' | 'past_due';
    currentPeriodStart?: string;
    currentPeriodEnd?: string;
    cancelAtPeriodEnd?: boolean;
  };
  usage: {
    linksUsed: number;
    linksLimit: number;
    portfoliosUsed: number;
    portfoliosLimit: number;
    storageUsed: number; // MB
    storageLimit: number; // MB
  };
}
```

### POST /api/subscription/checkout
**Stripe Checkout セッション作成**

```typescript
// リクエスト
interface CreateCheckoutRequest {
  planType: 'pro';
  successUrl: string;
  cancelUrl: string;
}

// レスポンス
interface CreateCheckoutResponse {
  checkoutUrl: string;
  sessionId: string;
}
```

### POST /api/subscription/cancel
**サブスクリプション解約**

```typescript
// レスポンス
interface CancelSubscriptionResponse {
  subscription: {
    status: 'canceled';
    cancelAtPeriodEnd: true;
    currentPeriodEnd: string;
  };
}
```

---

## 9. 🤖 AI API

### POST /api/ai/generate-bio
**自己紹介文生成**

```typescript
// リクエスト
interface GenerateBioRequest {
  name: string;
  title?: string;
  skills?: string[];
  interests?: string[];
  tone?: 'professional' | 'casual' | 'creative';
  language?: 'ja' | 'en';
}

// レスポンス
interface GenerateBioResponse {
  suggestions: string[];
  usage: {
    tokensUsed: number;
    remainingQuota: number;
  };
}
```

### POST /api/ai/generate-seo
**SEO最適化提案**

```typescript
// リクエスト
interface GenerateSEORequest {
  profileData: {
    name: string;
    title?: string;
    bio?: string;
    skills?: string[];
  };
  targetKeywords?: string[];
}

// レスポンス
interface GenerateSEOResponse {
  suggestions: {
    title: string[];
    description: string[];
    keywords: string[];
  };
}
```

---

## 10. 🔧 ユーティリティAPI

### GET /api/health
**ヘルスチェック**

```typescript
// レスポンス
interface HealthResponse {
  status: 'ok' | 'error';
  timestamp: string;
  version: string;
  services: {
    database: 'ok' | 'error';
    storage: 'ok' | 'error';
    ai: 'ok' | 'error';
  };
}
```

### POST /api/contact
**お問い合わせ**

```typescript
// リクエスト
interface ContactRequest {
  name: string;
  email: string;
  subject: string;
  message: string;
  type: 'support' | 'feedback' | 'bug' | 'feature';
}

// レスポンス
interface ContactResponse {
  ticketId: string;
  message: string;
}
```

---

## 11. 🚨 エラーコード

### 認証エラー
- `AUTH_INVALID_CREDENTIALS` - 認証情報が無効
- `AUTH_EMAIL_NOT_VERIFIED` - メール未認証
- `AUTH_TOKEN_EXPIRED` - トークン期限切れ
- `AUTH_INSUFFICIENT_PERMISSIONS` - 権限不足

### バリデーションエラー
- `VALIDATION_REQUIRED_FIELD` - 必須フィールド未入力
- `VALIDATION_INVALID_EMAIL` - メール形式が無効
- `VALIDATION_PASSWORD_TOO_WEAK` - パスワードが弱い
- `VALIDATION_USERNAME_TAKEN` - ユーザー名が使用済み
- `VALIDATION_FILE_TOO_LARGE` - ファイルサイズ超過
- `VALIDATION_INVALID_FILE_TYPE` - ファイル形式が無効

### ビジネスロジックエラー
- `LIMIT_LINKS_EXCEEDED` - リンク数上限超過
- `LIMIT_PORTFOLIOS_EXCEEDED` - ポートフォリオ数上限超過
- `LIMIT_STORAGE_EXCEEDED` - ストレージ容量超過
- `SUBSCRIPTION_REQUIRED` - サブスクリプション必須

### システムエラー
- `INTERNAL_SERVER_ERROR` - サーバー内部エラー
- `DATABASE_CONNECTION_ERROR` - DB接続エラー
- `STORAGE_UPLOAD_ERROR` - ファイルアップロードエラー
- `AI_SERVICE_UNAVAILABLE` - AIサービス利用不可

---

## 12. 📝 レート制限

### 制限値
- **認証API**: 5回/分
- **ファイルアップロード**: 10回/時間
- **AI API**: 20回/日（無料版）、100回/日（Pro版）
- **その他API**: 100回/分

### レスポンスヘッダー
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
```

---

## 13. 🔒 セキュリティ

### CORS設定
```javascript
// 許可ドメイン
const allowedOrigins = [
  'https://showya.app',
  'https://staging.showya.app',
  'http://localhost:3000' // 開発環境のみ
];
```

### CSP設定
```
Content-Security-Policy: default-src 'self'; 
  script-src 'self' 'unsafe-inline' *.vercel.app;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: *.supabase.co;
  connect-src 'self' *.supabase.co;
```

### 入力サニタイゼーション
- HTMLタグの除去
- SQLインジェクション対策
- XSS対策
- ファイルアップロード検証

---

## 14. 🔄 Bulk操作API

### POST /api/social-links/bulk
**複数リンクの一括操作**

```typescript
// リクエスト
interface BulkSocialLinksRequest {
  operations: {
    type: 'create' | 'update' | 'delete';
    id?: string; // update/deleteの場合必須
    data?: Partial<CreateSocialLinkRequest>; // create/updateの場合
  }[];
}

// レスポンス
interface BulkSocialLinksResponse {
  results: {
    operation: string;
    success: boolean;
    data?: SocialLinkWithStats;
    error?: string;
  }[];
  summary: {
    total: number;
    successful: number;
    failed: number;
  };
}
```

### POST /api/portfolios/bulk
**複数ポートフォリオの一括操作**

```typescript
// リクエスト
interface BulkPortfoliosRequest {
  operations: {
    type: 'create' | 'update' | 'delete';
    id?: string;
    data?: Partial<CreatePortfolioRequest>;
  }[];
}

// レスポンス
interface BulkPortfoliosResponse {
  results: {
    operation: string;
    success: boolean;
    data?: Portfolio;
    error?: string;
  }[];
  summary: {
    total: number;
    successful: number;
    failed: number;
  };
}
```

### POST /api/bulk/import
**データ一括インポート**

```typescript
// リクエスト
interface BulkImportRequest {
  format: 'json' | 'csv';
  data: {
    socialLinks?: CreateSocialLinkRequest[];
    portfolios?: CreatePortfolioRequest[];
  };
  options?: {
    skipDuplicates?: boolean;
    validateOnly?: boolean;
  };
}

// レスポンス
interface BulkImportResponse {
  importId: string;
  status: 'processing' | 'completed' | 'failed';
  summary: {
    totalRecords: number;
    processedRecords: number;
    successfulRecords: number;
    failedRecords: number;
  };
  errors?: {
    row: number;
    field: string;
    message: string;
  }[];
}
```

### GET /api/bulk/import/[importId]
**インポート状況確認**

```typescript
// レスポンス
interface ImportStatusResponse {
  importId: string;
  status: 'processing' | 'completed' | 'failed';
  progress: number; // 0-100
  summary: {
    totalRecords: number;
    processedRecords: number;
    successfulRecords: number;
    failedRecords: number;
  };
  startedAt: string;
  completedAt?: string;
  downloadUrl?: string; // エラーレポートのダウンロードURL
}
```

### GET /api/export/profile
**プロフィールデータエクスポート**

```typescript
// クエリパラメータ
interface ExportProfileQuery {
  format: 'json' | 'csv' | 'pdf';
  include?: ('social_links' | 'portfolios' | 'analytics')[];
  dateRange?: {
    start: string;
    end: string;
  };
}

// レスポンス
interface ExportProfileResponse {
  exportId: string;
  status: 'processing' | 'ready';
  downloadUrl?: string;
  expiresAt: string;
  format: string;
  fileSize?: number;
}
```

### GET /api/export/analytics
**分析データエクスポート**

```typescript
// クエリパラメータ
interface ExportAnalyticsQuery {
  format: 'json' | 'csv' | 'xlsx';
  metrics: ('clicks' | 'views' | 'conversions')[];
  groupBy?: 'day' | 'week' | 'month';
  dateRange: {
    start: string;
    end: string;
  };
}

// レスポンス
interface ExportAnalyticsResponse {
  exportId: string;
  status: 'processing' | 'ready';
  downloadUrl?: string;
  expiresAt: string;
  recordCount: number;
  columns: string[];
}
```

### POST /api/export/backup
**完全バックアップ作成**

```typescript
// リクエスト
interface CreateBackupRequest {
  includeAnalytics?: boolean;
  includeFiles?: boolean;
  format: 'json' | 'zip';
  encryption?: {
    enabled: boolean;
    password?: string;
  };
}

// レスポンス
interface CreateBackupResponse {
  backupId: string;
  status: 'processing' | 'ready';
  downloadUrl?: string;
  expiresAt: string;
  fileSize?: number;
  checksum?: string;
}
```

### GET /api/export/[exportId]
**エクスポート状況確認**

```typescript
// レスポンス
interface ExportStatusResponse {
  exportId: string;
  status: 'processing' | 'ready' | 'expired' | 'failed';
  progress: number; // 0-100
  downloadUrl?: string;
  expiresAt: string;
  fileSize?: number;
  error?: string;
  createdAt: string;
  completedAt?: string;
}
```

---

## 15. 📚 SDK・ライブラリ

### TypeScript SDK
```typescript
// インストール
npm install @showya/sdk

// 使用例
import { ShowyaClient } from '@showya/sdk';

const client = new ShowyaClient({
  apiKey: 'your-api-key',
  baseUrl: 'https://showya.app/api'
});

// プロフィール取得
const profile = await client.profiles.get('username');

// リンク追加
const link = await client.socialLinks.create({
  platform: 'twitter',
  url: 'https://twitter.com/username'
});
```

### Webhook
```typescript
// Stripe Webhook
POST /api/webhooks/stripe

// イベント例
interface StripeWebhookEvent {
  type: 'customer.subscription.created' | 'customer.subscription.deleted';
  data: {
    object: {
      id: string;
      customer: string;
      status: string;
    };
  };
}
```