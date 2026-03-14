# MVP 設計書

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [リポジトリ構成](#2-リポジトリ構成)
3. [データベース設計](#3-データベース設計postgresql)
4. [API 設計](#4-api-設計go--gin)
5. [ディレクトリ構成](#5-ディレクトリ構成)
6. [ローカル開発環境](#6-ローカル開発環境)
7. [環境変数](#7-環境変数)
8. [セキュリティ方針](#8-セキュリティ方針)

---

## 1. アーキテクチャ概要

### 1.1 技術スタック

| レイヤー | Phase 1（ローカル） | Phase 2（本番） |
|---|---|---|
| Frontend | Next.js `localhost:3000`（通常サーバー） | Vercel（無料・SSR 完全対応） |
| Backend | Go / Gin `localhost:8080`（Docker） | Render（Docker コンテナ・$7/月〜） |
| Database | PostgreSQL（Docker）`:5432` | Supabase PostgreSQL（無料枠 500MB） |
| 画像 | Go ローカルディスク `/uploads/` | Supabase Storage（無料枠 1GB） |
| 認証 | Go 自前 JWT | Go 自前 JWT（変更なし） |
| AI | Gemini 2.5 Flash API | Gemini 2.5 Flash API（変更なし） |
| メール確認 | Mailpit（Docker） | Supabase Auth メール or SendGrid |

### 1.2 Phase 構成

```
Phase 1: ローカル開発（アプリケーションを完成させる）
  - Next.js は npm run dev で通常起動
  - PostgreSQL + Go は Docker で起動
  - メール確認は Mailpit（Docker）

Phase 2: 本番リリース
  - Vercel（Frontend）
  - Render（Backend・Docker コンテナ）
  - Supabase（PostgreSQL + Storage）
```

### 1.3 ユーザー種別と権限

| 種別 | DB 管理 | 利用可能機能 |
|---|---|---|
| ゲストユーザー | なし（フロントのみ状態管理） | ボード閲覧・GPS 判定・音声発話・AI レコメンド |
| ログインユーザー | あり（users テーブル） | ゲスト機能 ＋ カスタムロケーション追加・カード編集・画像アップロード |

> **ゲストユーザーの状態管理**
> 選択カードや一時的なボード状態はフロントエンドの **Zustand**（メモリ）で管理する。
> Next.js App Router 環境では `"use client"` コンポーネント内でのみ Zustand を使用する。
> 今回のボード画面はインタラクティブ操作が中心のため Client Components がほとんどになり、Zustand は適切な選択。
> DB への読み書きは行わない。

### 1.4 Phase 移行の設計方針

`storage.ImageStorage` インターフェースで抽象化することで、
Phase 1 → 2 の移行は実装の差し替えと環境変数の変更のみで完結する。

```go
// backend/internal/storage/storage.go
type ImageStorage interface {
    Upload(ctx context.Context, key string, data []byte, contentType string) (url string, err error)
    Delete(ctx context.Context, key string) error
    GetPublicURL(key string) string
}

// Phase 1: LocalStorage 実装（/uploads/ に保存・http.FileServer で配信）
// Phase 2: SupabaseStorage 実装（Supabase Storage SDK）
// 環境変数 STORAGE_BACKEND=local | supabase で切り替え
```

---

## 2. リポジトリ構成

Git サブモジュール構成を採用する。ルートリポジトリ配下に backend・frontend を別リポジトリとして管理する。

```
tsu_hack/                  # ルートリポジトリ（サブモジュール管理）
├── backend/               # サブモジュール: tsu_hackB
├── frontend/              # サブモジュール: tsu_hackF
└── README.md
```

### tsu_hackB（バックエンド）

```
tsu_hackB/
├── cmd/
├── internal/
├── migrations/
├── supabase/              # Supabase CLIマイグレーション・シードをここに集約
│   ├── config.toml
│   ├── migrations/
│   └── seed.sql
├── Dockerfile
├── go.mod
└── .env.example
```

> **supabase/ の配置について**
> マイグレーション実行は Go サーバーの起動と紐づくため、
> DB スキーマ管理とバックエンドを同じリポジトリ（tsu_hackB）で管理するのが一貫性があり適切。

### tsu_hackF（フロントエンド）

```
tsu_hackF/
├── src/
├── public/
├── next.config.ts
├── tailwind.config.ts
└── package.json
```

---

## 3. データベース設計（PostgreSQL）

### 3.1 テーブル一覧

| テーブル名 | 役割 | 備考 |
|---|---|---|
| `users` | ログインユーザー管理 | ゲストは DB 登録なし |
| `locations` | 共有ロケーションマスター | コンビニ/病院/カフェをシード投入 |
| `cards` | カードマスター（単語カード） | `created_by=NULL` がシステム共有カード |
| `location_cards` | 共有ロケーション × カード | デフォルトボードのカード紐付け |
| `user_locations` | ユーザー独自ロケーション | GPS 座標 + 判定半径 |
| `user_location_cards` | ユーザーロケーション × カード | カスタムボードのカード紐付け |

> **助詞カードは DB に登録しない**
> 「を・に・は・が」等の助詞は `cards` テーブルに登録しない。
> ユーザーが選択した単語配列を Gemini 2.5 Flash に渡し、自然な文章として補完する。
> カードの組み合わせ爆発を防ぎ、UX の一貫性を保つ。

### 3.2 DDL

#### ① users

```sql
CREATE TABLE users (
  id            UUID         NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  email         VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  display_name  VARCHAR(100) NOT NULL,
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

#### ② locations（共有ロケーションマスター）

```sql
CREATE TABLE locations (
  id          UUID             NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  name        VARCHAR(100)     NOT NULL,            -- 例: コンビニ
  description TEXT,
  latitude    DOUBLE PRECISION,                     -- 基準緯度（距離判定用）
  longitude   DOUBLE PRECISION,                     -- 基準経度
  radius_m    INTEGER          NOT NULL DEFAULT 200, -- 判定半径（メートル）
  is_default  BOOLEAN          NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ      NOT NULL DEFAULT NOW()
);

-- 初期シードデータ
INSERT INTO locations (name, radius_m) VALUES
  ('コンビニ', 100),
  ('病院',     200),
  ('カフェ',   100);
```

#### ③ cards（カードマスター）

```sql
CREATE TABLE cards (
  id         UUID         NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  label      VARCHAR(100) NOT NULL,            -- 表示テキスト（例: おにぎり）
  image_url  TEXT,                             -- 画像URL（Supabase Storage or ローカル）
  emoji      VARCHAR(10),                      -- image_url がない場合の絵文字
  category   VARCHAR(50),                      -- 日常 / 食べ物 / 動詞 など
  is_daily   BOOLEAN      NOT NULL DEFAULT false, -- 日常カードフラグ
  created_by UUID         REFERENCES users(id) ON DELETE SET NULL, -- NULL はシステム共有
  created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- 日常カードのシード
INSERT INTO cards (label, emoji, is_daily) VALUES
  ('こんにちは', '👋', true),
  ('ありがとう', '🙏', true),
  ('すみません', '🙇', true),
  ('はい',       '✅', true),
  ('いいえ',     '❌', true);
```

#### ④ location_cards（共有ロケーション × カード）

```sql
CREATE TABLE location_cards (
  id          UUID        NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  location_id UUID        NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
  card_id     UUID        NOT NULL REFERENCES cards(id)     ON DELETE CASCADE,
  sort_order  INTEGER     NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (location_id, card_id)
);
```

#### ⑤ user_locations（ユーザー独自ロケーション）

```sql
CREATE TABLE user_locations (
  id         UUID             NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id    UUID             NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name       VARCHAR(100)     NOT NULL,       -- 例: ピアノ教室
  latitude   DOUBLE PRECISION NOT NULL,
  longitude  DOUBLE PRECISION NOT NULL,
  radius_m   INTEGER          NOT NULL DEFAULT 100,
  created_at TIMESTAMPTZ      NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ      NOT NULL DEFAULT NOW()
);
```

#### ⑥ user_location_cards（ユーザーロケーション × カード）

```sql
CREATE TABLE user_location_cards (
  id               UUID        NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  user_location_id UUID        NOT NULL REFERENCES user_locations(id) ON DELETE CASCADE,
  card_id          UUID        NOT NULL REFERENCES cards(id)          ON DELETE CASCADE,
  sort_order       INTEGER     NOT NULL DEFAULT 0,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (user_location_id, card_id)
);
```

### 3.3 インデックス

```sql
-- ロケーション距離判定（緯度経度での概算フィルタ）
CREATE INDEX idx_locations_latlng         ON locations(latitude, longitude);
CREATE INDEX idx_user_locations_user      ON user_locations(user_id);
CREATE INDEX idx_user_locations_latlng    ON user_locations(latitude, longitude);

-- カード取得の高速化
CREATE INDEX idx_location_cards_location  ON location_cards(location_id, sort_order);
CREATE INDEX idx_user_location_cards_ul   ON user_location_cards(user_location_id, sort_order);
CREATE INDEX idx_cards_is_daily           ON cards(is_daily);
CREATE INDEX idx_cards_created_by         ON cards(created_by);
```

---

## 4. API 設計（Go / Gin）

### 4.1 共通仕様

| 項目 | 内容 |
|---|---|
| Base URL | `/api/v1` |
| 認証ヘッダー | `Authorization: Bearer <access_token>` |
| Content-Type | `application/json`（画像アップロードのみ `multipart/form-data`） |
| エラー形式 | `{ "error": "メッセージ", "code": "ERROR_CODE" }` |
| ゲスト可 | 表中「不要」のエンドポイントは未認証で呼び出し可能 |

### 4.2 認証 API（`/api/v1/auth`）

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| POST | `/auth/signup` | 不要 | 新規ユーザー登録（email / password / display_name） |
| POST | `/auth/login` | 不要 | ログイン → access_token / refresh_token 返却 |
| POST | `/auth/refresh` | 不要 | refresh_token で access_token 再発行 |
| POST | `/auth/logout` | 必要 | refresh_token を無効化（DB のトークン削除） |
| GET  | `/auth/me` | 必要 | ログイン中ユーザー情報取得 |

**POST /auth/login レスポンス例**

```json
{
  "access_token":  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "token_type":    "bearer",
  "expires_in":    900
}
```

### 4.3 ロケーション API（`/api/v1/locations`）

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| GET    | `/locations/nearby`           | 不要 | 現在地(lat,lng)から半径内のロケーションを距離順で返す |
| GET    | `/locations`                  | 不要 | 共有ロケーション一覧（デフォルトボード） |
| GET    | `/locations/:id/cards`        | 不要 | 共有ロケーションのカード一覧 |
| GET    | `/user/locations`             | 必要 | ユーザー独自ロケーション一覧 |
| POST   | `/user/locations`             | 必要 | 独自ロケーション追加 |
| PUT    | `/user/locations/:id`         | 必要 | 独自ロケーション更新 |
| DELETE | `/user/locations/:id`         | 必要 | 独自ロケーション削除 |
| GET    | `/user/locations/:id/cards`   | 必要 | ユーザーロケーションのカード一覧 |

**GET /locations/nearby クエリパラメータ例**

```
GET /api/v1/locations/nearby?lat=33.2500&lng=131.6100&radius_m=500
```

```json
[
  {
    "id":          "uuid-xxxx",
    "name":        "コンビニ",
    "type":        "shared",
    "distance_m":  42,
    "cards_count": 12
  }
]
```

> **Haversine 公式について**
> Haversine 公式とは、球面上の 2 地点間の距離を緯度・経度から計算する公式。
> `GET /locations/nearby` のバックエンド実装では、まず DB クエリで
> `±0.01度` の概算範囲フィルタをかけて候補を絞り込み、
> 取得した候補に対して Go のサービス層で Haversine 公式による正確な距離計算を行い、
> `radius_m` 以内のロケーションのみ距離昇順で返す。

### 4.4 カード API（`/api/v1/cards`）

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| GET    | `/cards/daily`                              | 不要 | 日常カード一覧（is_daily=true） |
| POST   | `/user/cards`                               | 必要 | ユーザー独自カード作成 |
| POST   | `/user/locations/:id/cards`                 | 必要 | ロケーションにカードを追加（共有カードも可） |
| DELETE | `/user/locations/:id/cards/:card_id`        | 必要 | ロケーションからカードを削除 |
| PUT    | `/user/locations/:id/cards/reorder`         | 必要 | カード表示順の一括更新 |

### 4.5 画像アップロード API

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| POST | `/upload/image` | 必要 | 画像アップロード（multipart/form-data）→ 保存先 URL を返す |

> **Phase 1 / Phase 2 の動作**
> - Phase 1: Go サーバーが `/uploads/` に保存し `/uploads/{key}` で配信
> - Phase 2: `storage.ImageStorage` の実装が Supabase Storage に切り替わり公開 URL を返す
> - API のインターフェースは両 Phase で変わらない

**リクエスト / レスポンス例**

```
POST /api/v1/upload/image
Content-Type: multipart/form-data
Body: file=<image binary>
```

```json
{
  "url": "https://xxxx.supabase.co/storage/v1/object/public/cards/uuid.jpg",
  "key": "cards/uuid.jpg"
}
```

### 4.6 AI レコメンド API

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| POST | `/ai/recommend` | 不要 | 選択単語配列 → 自然な文章候補を返す（Gemini 2.5 Flash） |

**リクエスト / レスポンス例**

```json
// Request
{
  "words":         ["おにぎり", "ください"],
  "location_name": "コンビニ"
}

// Response
{
  "suggestions": [
    "おにぎりをください。",
    "おにぎりを1つください。"
  ],
  "latency_ms": 280
}
```

> **レイテンシ対策**
> フロントエンドはカード選択後 500ms のデバウンスを経てリクエスト送信。
> AI 生成中も元の単語列をそのまま発音ボタンで発話できる設計を維持し、ユーザーを待たせない。

---

## 5. ディレクトリ構成

### 5.1 tsu_hackB（バックエンド）

```
tsu_hackB/
├── cmd/
│   └── server/
│       └── main.go                   # エントリーポイント
│
├── internal/
│   ├── config/
│   │   └── config.go                 # 環境変数読み込み（godotenv）
│   │
│   ├── db/
│   │   └── db.go                     # PostgreSQL 接続（pgx/v5）
│   │
│   ├── handler/                      # Gin ハンドラ層（薄く保つ）
│   │   ├── auth.go                   # POST /auth/*
│   │   ├── location.go               # GET /locations/*
│   │   ├── card.go                   # GET,POST /cards/*
│   │   ├── user_location.go          # CRUD /user/locations/*
│   │   ├── upload.go                 # POST /upload/image
│   │   └── ai.go                     # POST /ai/recommend
│   │
│   ├── middleware/
│   │   ├── auth.go                   # JWT 検証ミドルウェア
│   │   └── cors.go                   # CORS 設定
│   │
│   ├── service/                      # ビジネスロジック層
│   │   ├── auth_service.go           # JWT 発行・検証・bcrypt
│   │   ├── location_service.go       # Haversine 距離計算
│   │   ├── card_service.go           # カード CRUD
│   │   └── ai_service.go             # Gemini 2.5 Flash API 呼び出し
│   │
│   ├── storage/
│   │   ├── storage.go                # ImageStorage インターフェース定義
│   │   ├── local.go                  # LocalStorage（Phase 1）
│   │   └── supabase.go               # SupabaseStorage（Phase 2）
│   │
│   ├── model/                        # DB モデル・リクエスト/レスポンス型
│   │   ├── user.go
│   │   ├── location.go
│   │   ├── card.go
│   │   └── ai.go
│   │
│   └── router/
│       └── router.go                 # Gin ルーティング定義
│
├── migrations/                       # SQL マイグレーションファイル
│   ├── 001_create_users.sql
│   ├── 002_create_locations.sql
│   ├── 003_create_cards.sql
│   ├── 004_create_location_cards.sql
│   ├── 005_create_user_locations.sql
│   ├── 006_create_user_location_cards.sql
│   └── 007_seed_initial_data.sql     # コンビニ/病院/カフェ + 日常カード
│
├── supabase/                         # Supabase CLI 管理（tsu_hackB に集約）
│   ├── config.toml
│   ├── migrations/                   # Supabase 用マイグレーション
│   └── seed.sql
│
├── Dockerfile
├── docker-compose.yml                # PostgreSQL + Go + Mailpit（ローカル開発用）
├── go.mod
├── go.sum
└── .env.example
```

### 5.2 tsu_hackF（フロントエンド）

```
tsu_hackF/
├── src/
│   ├── app/                              # App Router
│   │   ├── layout.tsx                    # ルートレイアウト
│   │   ├── page.tsx                      # ホーム画面（GPS 取得・ボード選択）
│   │   ├── board/
│   │   │   └── [locationId]/
│   │   │       └── page.tsx              # ボード画面（コミュニケーション）
│   │   ├── edit/
│   │   │   ├── locations/page.tsx        # ロケーション追加・編集
│   │   │   └── cards/page.tsx            # カード追加・編集
│   │   └── auth/
│   │       ├── login/page.tsx
│   │       └── register/page.tsx
│   │
│   ├── components/
│   │   ├── board/
│   │   │   ├── CardGrid.tsx              # カードグリッド（大ボタン）
│   │   │   ├── CardItem.tsx              # 単一カード
│   │   │   ├── SelectedList.tsx          # 選択リスト（ストック箱）
│   │   │   ├── AiSuggestion.tsx          # AI レコメンド表示領域
│   │   │   └── SpeakButton.tsx           # 発音ボタン（Web Speech API）
│   │   ├── location/
│   │   │   ├── LocationFinder.tsx        # GPS 取得 → 近隣ロケーション提案
│   │   │   └── LocationChanger.tsx       # ロケーション変更ボタン
│   │   ├── edit/
│   │   │   ├── CardEditor.tsx            # カード追加フォーム
│   │   │   └── ImageUploader.tsx         # 画像アップロード
│   │   └── ui/                           # 汎用 UI コンポーネント
│   │       ├── Button.tsx
│   │       ├── LoadingSpinner.tsx
│   │       └── Modal.tsx
│   │
│   ├── hooks/
│   │   ├── useGeolocation.ts             # Geolocation API フック
│   │   ├── useSpeech.ts                  # Web Speech API（TTS）フック
│   │   ├── useAiRecommend.ts             # AI レコメンド（デバウンス 500ms）
│   │   └── useAuth.ts                    # 認証状態管理
│   │
│   ├── lib/
│   │   ├── api.ts                        # API クライアント（fetch wrapper）
│   │   ├── auth.ts                       # トークン管理（httpOnly Cookie）
│   │   └── geo.ts                        # 距離計算ユーティリティ（クライアント側）
│   │
│   ├── store/
│   │   └── boardStore.ts                 # Zustand: 選択カード・AI 候補状態
│   │                                     # ※ "use client" コンポーネント内でのみ使用
│   │
│   └── types/
│       ├── location.ts
│       ├── card.ts
│       └── auth.ts
│
├── public/icons/                         # デフォルト絵記号画像
├── next.config.ts
├── tailwind.config.ts
└── package.json
```

---

## 6. ローカル開発環境

### 6.1 起動方法

Next.js はローカルで通常起動、PostgreSQL と Go は Docker で起動する。

```bash
# 1. PostgreSQL + Go + Mailpit を Docker で起動（tsu_hackB 内）
docker compose up -d

# 2. マイグレーション実行
docker compose exec backend go run ./cmd/migrate

# 3. Next.js をローカルで起動（tsu_hackF 内）
npm run dev
```

### 6.2 docker-compose.yml（tsu_hackB）

PostgreSQL・Go バックエンド・Mailpit の 3 サービスのみ含める。
Next.js は含めない。

```yaml
services:
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: aac_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - pgdata:/var/lib/postgresql/data

  backend:
    build: .
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      - db
    volumes:
      - ./uploads:/app/uploads    # 画像の永続化（Phase 1）

  mailpit:
    image: axllent/mailpit
    ports:
      - "8025:8025"   # Web UI（メール確認）
      - "1025:1025"   # SMTP（バックエンドからの送信先）

volumes:
  pgdata:
```

### 6.3 Mailpit について

Mailpit は Docker イメージ 1 行で起動できるローカル用メールキャッチャー。
バックエンドのメール送信先 SMTP を `localhost:1025` に向けるだけで、
送信されたメールをブラウザ（`http://localhost:8025`）で確認できる。

```
# backend/.env の SMTP 設定（Phase 1 ローカル用）
SMTP_HOST=localhost
SMTP_PORT=1025
```

追加設定・アカウント登録は一切不要で、認証メールのテストに使える。

---

## 7. 環境変数

### 7.1 バックエンド（`backend/.env`）

```env
# サーバー
PORT=8080
ENV=development   # development | production

# データベース
# Phase 1: Docker PostgreSQL
# Phase 2: Supabase 接続文字列に変更
DATABASE_URL=postgres://postgres:password@localhost:5432/aac_dev

# JWT
JWT_SECRET=your-secret-key-min-32chars
JWT_ACCESS_EXPIRES_MIN=15
JWT_REFRESH_EXPIRES_DAYS=7

# 画像ストレージ切り替え
STORAGE_BACKEND=local            # local | supabase
UPLOAD_DIR=./uploads             # Phase 1: ローカルディスク保存先
BASE_URL=http://localhost:8080   # Phase 1: 画像配信ベース URL

# Supabase（Phase 2 で設定）
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_STORAGE_BUCKET=cards

# Gemini
GEMINI_API_KEY=AIzaSy...

# SMTP（Phase 1: Mailpit / Phase 2: SendGrid 等）
SMTP_HOST=localhost
SMTP_PORT=1025

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:3000,https://your-app.vercel.app
```

### 7.2 フロントエンド（`frontend/.env.local`）

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080/api/v1
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 8. セキュリティ方針

### 8.1 認証・JWT

- パスワード: bcrypt（コスト係数 12 以上）
- アクセストークン有効期限: 15 分
- リフレッシュトークン有効期限: 7 日
- リフレッシュトークンは DB に保存し、ログアウト時に削除（無効化）
- JWT の秘密鍵は環境変数で管理（ハードコード禁止）

### 8.2 画像アップロード

- Content-Type 検証（`image/jpeg` / `image/png` / `image/webp` のみ許可）
- ファイルサイズ上限: 5MB
- 保存時にファイル名を UUID にリネーム（パストラバーサル防止）
- Phase 2: Supabase Storage の RLS（Row Level Security）でユーザーごとにアクセス制限

### 8.3 API 共通

- 全エンドポイントに CORS 設定（許可オリジンを環境変数で管理）
- Gin の Rate Limiting ミドルウェアで連続リクエスト制限
- DB クエリはプリペアドステートメントのみ使用（SQL インジェクション防止）
- `Authorization` ヘッダーは HTTPS 経由でのみ送信（Phase 2: Render + Vercel はデフォルト HTTPS）
