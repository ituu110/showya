# 🔧 技術仕様書

## プロジェクト名：showya

---

## 1. 🏗️ システム構成

### アーキテクチャ概要
```
[ユーザー] → [Vercel CDN] → [Next.js App] → [Supabase] → [PostgreSQL]
                ↓
           [画像・動画ストレージ]
```

### 技術スタック詳細

| レイヤー | 技術 | バージョン | 理由 |
|----------|------|------------|------|
| フロントエンド | React (Next.js) | 18.x / 14.x | コンポーネントベース、SSG/SSR、SEO最適化 |
| ランタイム | Node.js | 18.x+ | Next.js実行環境、API Routes |
| スタイリング | TailwindCSS | 3.x | 高速開発、軽量 |
| バックエンド | Supabase + Next.js API | Latest | 認証・DB・ストレージ統合 + サーバーサイド処理 |
| データベース | PostgreSQL | 15.x | リレーショナルDB |
| ホスティング | Vercel | Latest | Next.js最適化、CDN |
| 画像最適化 | Next.js Image | Built-in | WebP変換、遅延読み込み |
| 多言語化 | next-i18next | 13.x | 静的生成対応 |
| SEO | next-seo | 6.x | メタタグ管理 |

---

## 2. 🗄️ データベース設計

### ERD概要
```
users (1) ←→ (n) profiles (1) ←→ (n) social_links
  ↓                ↓
(1:n)            (1:n)
  ↓                ↓
subscriptions   portfolios (1) ←→ (n) portfolio_tags
```

### テーブル定義

#### users テーブル
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  email_verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### profiles テーブル
```sql
CREATE TABLE profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  username VARCHAR(50) UNIQUE NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  title VARCHAR(100),
  bio TEXT CHECK (LENGTH(bio) <= 500),
  avatar_url TEXT,
  custom_domain VARCHAR(100),
  seo_title VARCHAR(60),
  seo_description VARCHAR(160),
  is_public BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### social_links テーブル
```sql
CREATE TABLE social_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  profile_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
  platform VARCHAR(50) NOT NULL, -- 'twitter', 'instagram', 'github', 'custom'
  url TEXT NOT NULL,
  title VARCHAR(100), -- カスタムリンクのタイトル
  icon_url TEXT, -- カスタムアイコン
  display_order INTEGER DEFAULT 0,
  click_count INTEGER DEFAULT 0,
  last_clicked_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### portfolios テーブル
```sql
CREATE TABLE portfolios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  profile_id UUID REFERENCES profiles(id) ON DELETE CASCADE,
  title VARCHAR(100) NOT NULL,
  subtitle VARCHAR(200),
  description TEXT,
  type VARCHAR(20) NOT NULL, -- 'image', 'video', 'website'
  file_url TEXT, -- 画像・動画のURL
  website_url TEXT, -- ウェブサイトのURL
  thumbnail_url TEXT,
  display_order INTEGER DEFAULT 0,
  is_featured BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### portfolio_tags テーブル
```sql
CREATE TABLE portfolio_tags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id UUID REFERENCES portfolios(id) ON DELETE CASCADE,
  tag_name VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### subscriptions テーブル
```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  plan_type VARCHAR(20) DEFAULT 'free', -- 'free', 'pro'
  stripe_customer_id VARCHAR(100),
  stripe_subscription_id VARCHAR(100),
  status VARCHAR(20) DEFAULT 'active', -- 'active', 'canceled', 'past_due'
  current_period_start TIMESTAMP,
  current_period_end TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### インデックス設計

#### パフォーマンス最適化インデックス
```sql
-- ユーザー検索用
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- プロフィール検索用
CREATE UNIQUE INDEX idx_profiles_username ON profiles(username);
CREATE INDEX idx_profiles_user_id ON profiles(user_id);
CREATE INDEX idx_profiles_is_public ON profiles(is_public) WHERE is_public = true;
CREATE INDEX idx_profiles_updated_at ON profiles(updated_at);

-- ソーシャルリンク用
CREATE INDEX idx_social_links_profile_id ON social_links(profile_id);
CREATE INDEX idx_social_links_display_order ON social_links(profile_id, display_order);
CREATE INDEX idx_social_links_platform ON social_links(platform);
CREATE INDEX idx_social_links_click_count ON social_links(click_count DESC);

-- ポートフォリオ用
CREATE INDEX idx_portfolios_profile_id ON portfolios(profile_id);
CREATE INDEX idx_portfolios_display_order ON portfolios(profile_id, display_order);
CREATE INDEX idx_portfolios_type ON portfolios(type);
CREATE INDEX idx_portfolios_is_featured ON portfolios(is_featured) WHERE is_featured = true;
CREATE INDEX idx_portfolios_created_at ON portfolios(created_at DESC);

-- ポートフォリオタグ用
CREATE INDEX idx_portfolio_tags_portfolio_id ON portfolio_tags(portfolio_id);
CREATE INDEX idx_portfolio_tags_tag_name ON portfolio_tags(tag_name);
CREATE INDEX idx_portfolio_tags_composite ON portfolio_tags(portfolio_id, tag_name);

-- サブスクリプション用
CREATE UNIQUE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe_customer ON subscriptions(stripe_customer_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
CREATE INDEX idx_subscriptions_period_end ON subscriptions(current_period_end);
```

#### 全文検索インデックス
```sql
-- プロフィール全文検索用
CREATE INDEX idx_profiles_search ON profiles 
USING gin(to_tsvector('japanese', display_name || ' ' || COALESCE(title, '') || ' ' || COALESCE(bio, '')));

-- ポートフォリオ全文検索用
CREATE INDEX idx_portfolios_search ON portfolios 
USING gin(to_tsvector('japanese', title || ' ' || COALESCE(subtitle, '') || ' ' || COALESCE(description, '')));
```

### Row Level Security (RLS) ポリシー

#### RLS有効化
```sql
-- 全テーブルでRLSを有効化
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE social_links ENABLE ROW LEVEL SECURITY;
ALTER TABLE portfolios ENABLE ROW LEVEL SECURITY;
ALTER TABLE portfolio_tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
```

#### ユーザーポリシー
```sql
-- ユーザーは自分の情報のみアクセス可能
CREATE POLICY "Users can view own data" ON users
  FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can update own data" ON users
  FOR UPDATE USING (auth.uid() = id);

CREATE POLICY "Users can delete own data" ON users
  FOR DELETE USING (auth.uid() = id);
```

#### プロフィールポリシー
```sql
-- 公開プロフィールは誰でも閲覧可能、編集は所有者のみ
CREATE POLICY "Public profiles are viewable by everyone" ON profiles
  FOR SELECT USING (is_public = true);

CREATE POLICY "Users can view own profile" ON profiles
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own profile" ON profiles
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own profile" ON profiles
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own profile" ON profiles
  FOR DELETE USING (auth.uid() = user_id);
```

#### ソーシャルリンクポリシー
```sql
-- ソーシャルリンクは公開プロフィールと連動
CREATE POLICY "Social links viewable with public profile" ON social_links
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM profiles 
      WHERE profiles.id = social_links.profile_id 
      AND (profiles.is_public = true OR profiles.user_id = auth.uid())
    )
  );

CREATE POLICY "Users can manage own social links" ON social_links
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM profiles 
      WHERE profiles.id = social_links.profile_id 
      AND profiles.user_id = auth.uid()
    )
  );
```

#### ポートフォリオポリシー
```sql
-- ポートフォリオも公開プロフィールと連動
CREATE POLICY "Portfolios viewable with public profile" ON portfolios
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM profiles 
      WHERE profiles.id = portfolios.profile_id 
      AND (profiles.is_public = true OR profiles.user_id = auth.uid())
    )
  );

CREATE POLICY "Users can manage own portfolios" ON portfolios
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM profiles 
      WHERE profiles.id = portfolios.profile_id 
      AND profiles.user_id = auth.uid()
    )
  );
```

#### ポートフォリオタグポリシー
```sql
CREATE POLICY "Portfolio tags follow portfolio policy" ON portfolio_tags
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM portfolios 
      JOIN profiles ON profiles.id = portfolios.profile_id
      WHERE portfolios.id = portfolio_tags.portfolio_id 
      AND (profiles.is_public = true OR profiles.user_id = auth.uid())
    )
  );

CREATE POLICY "Users can manage own portfolio tags" ON portfolio_tags
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM portfolios 
      JOIN profiles ON profiles.id = portfolios.profile_id
      WHERE portfolios.id = portfolio_tags.portfolio_id 
      AND profiles.user_id = auth.uid()
    )
  );
```

#### サブスクリプションポリシー
```sql
-- サブスクリプション情報は所有者のみアクセス可能
CREATE POLICY "Users can view own subscription" ON subscriptions
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can update own subscription" ON subscriptions
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Service role can manage subscriptions" ON subscriptions
  FOR ALL USING (auth.role() = 'service_role');
```

---

## 3. 🔌 API設計

### 認証API
```
POST /api/auth/signup
POST /api/auth/signin
POST /api/auth/signout
POST /api/auth/reset-password
```

### プロフィールAPI
```
GET    /api/profiles/[username]     # 公開プロフィール取得
GET    /api/profiles/me             # 自分のプロフィール取得
PUT    /api/profiles/me             # プロフィール更新
DELETE /api/profiles/me             # プロフィール削除
```

### ソーシャルリンクAPI
```
GET    /api/social-links            # リンク一覧取得
POST   /api/social-links            # リンク追加
PUT    /api/social-links/[id]       # リンク更新
DELETE /api/social-links/[id]       # リンク削除
POST   /api/social-links/[id]/click # クリック数カウント
```

### ポートフォリオAPI
```
GET    /api/portfolios              # ポートフォリオ一覧
POST   /api/portfolios              # ポートフォリオ追加
PUT    /api/portfolios/[id]         # ポートフォリオ更新
DELETE /api/portfolios/[id]         # ポートフォリオ削除
POST   /api/portfolios/upload       # ファイルアップロード
```

### SEO API
```
GET    /api/seo/generate            # SEOタグ自動生成
PUT    /api/seo/update              # SEOタグ手動更新
```

---

## 4. 🎨 フロントエンド設計

### ページ構成
```
/                    # ランディングページ
/signup              # ユーザー登録
/signin              # ログイン
/dashboard           # ダッシュボード
/dashboard/profile   # プロフィール編集
/dashboard/links     # リンク管理
/dashboard/portfolio # ポートフォリオ管理
/dashboard/seo       # SEO設定
/dashboard/settings  # アカウント設定
/[username]          # 公開プロフィールページ
```

### コンポーネント設計
```
components/
├── layout/
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── Sidebar.tsx
├── forms/
│   ├── ProfileForm.tsx
│   ├── SocialLinkForm.tsx
│   └── PortfolioForm.tsx
├── ui/
│   ├── Button.tsx
│   ├── Input.tsx
│   ├── Modal.tsx
│   └── Card.tsx
└── profile/
    ├── ProfileHeader.tsx
    ├── SocialLinks.tsx
    └── PortfolioGrid.tsx
```

---

## 5. 🔒 セキュリティ実装

### 認証・認可
- Supabase Auth（JWT トークンベース）
- Row Level Security (RLS) 有効化
- セッション管理：httpOnly Cookie

### ファイルアップロード
```typescript
// ファイル検証ロジック
const validateFile = (file: File) => {
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'video/mp4'];
  const maxSize = file.type.startsWith('image/') ? 5 * 1024 * 1024 : 50 * 1024 * 1024;
  
  if (!allowedTypes.includes(file.type)) {
    throw new Error('不正なファイル形式です');
  }
  
  if (file.size > maxSize) {
    throw new Error('ファイルサイズが上限を超えています');
  }
};
```

### CSP設定
```javascript
// next.config.js
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline' *.vercel.app;
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data: *.supabase.co;
  media-src 'self' *.supabase.co;
  connect-src 'self' *.supabase.co;
`;
```

---

## 6. 📊 パフォーマンス最適化

### 画像最適化
```typescript
// next.config.js
module.exports = {
  images: {
    formats: ['image/webp', 'image/avif'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
};
```

### キャッシュ戦略
- 静的アセット：1年間キャッシュ
- API レスポンス：5分間キャッシュ
- プロフィールページ：ISR（1時間）

### バンドル最適化
```typescript
// next.config.js
module.exports = {
  experimental: {
    optimizeCss: true,
    optimizePackageImports: ['@heroicons/react'],
  },
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
};
```

---

## 7. 🚀 デプロイメント

### 環境構成
- **開発環境**: localhost:3000
- **ステージング環境**: staging.showya.app
- **本番環境**: showya.app

### CI/CD パイプライン
```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run build
      - run: npm run test
      - uses: amondnet/vercel-action@v20
```

### 環境変数
```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
OPENAI_API_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
```