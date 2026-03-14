# MVP 設計書

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [リポジトリ構成](#2-リポジトリ構成)
3. [データベース設計](#3-データベース設計mysql)
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
| Database | MySQL 8.0（Docker）`:3306` | Render MySQL アドオン |
| 画像ストレージ | Supabase Storage（クラウド・両 Phase 共通） | Supabase Storage（同左） |
| 認証 | Go 自前 JWT | Go 自前 JWT（変更なし） |
| AI | Gemini 2.5 Flash API | Gemini 2.5 Flash API（変更なし） |
| メール確認 | Mailpit（Docker） | SendGrid 等 |

> **Supabase は Storage のみ使用**
> DB 機能（PostgreSQL）は使用しない。
> `SUPABASE_URL` と `SUPABASE_SERVICE_KEY` があれば DB と無関係に Storage だけ利用できる。
> Phase 1・Phase 2 で同じ Supabase プロジェクトを使い回す。

### 1.2 Phase 構成

```
Phase 1: ローカル開発（アプリケーションを完成させる）
  - Next.js は npm run dev で通常起動
  - MySQL + Go + Mailpit は Docker で起動
  - Supabase Storage はクラウドにそのまま接続

Phase 2: 本番リリース
  - Vercel（Frontend）
  - Render（Backend・Docker コンテナ）
  - Render MySQL アドオン（DB）
  - Supabase Storage（画像・変更なし）
```

### 1.3 ユーザー種別と権限

| 種別 | DB 管理 | 利用可能機能 |
|---|---|---|
| ゲストユーザー | なし（フロントのみ状態管理） | ボード閲覧・GPS 判定・音声発話・AI レコメンド |
| ログインユーザー | あり（users テーブル） | ゲスト機能 ＋ カスタムロケーション追加・カード編集・画像アップロード |

> **ゲストユーザーの状態管理**
> 選択カードや一時的なボード状態はフロントエンドの **Zustand**（メモリ）で管理する。
> Next.js App Router 環境では `"use client"` コンポーネント内でのみ Zustand を使用する。
> ボード画面はインタラクティブ操作が中心のため Client Components がほとんどになり、Zustand は適切な選択。
> DB への読み書きは行わない。

### 1.4 画像アップロードのフロー

カード作成と画像アップロードを **1リクエストで完結** させる。
画像だけ上がってカード未作成という中途半端な状態が残らない。

```
① フロントが multipart/form-data で POST /user/cards を送信
       { file: <画像バイナリ>, label: "おにぎり", category: "食べ物", ... }
            ↓
② Go がリクエストから画像を取り出し Supabase Storage にアップロード
            ↓
③ Supabase Storage から公開 URL を取得
            ↓
④ label・category・image_url 等を MySQL の cards テーブルに INSERT
            ↓
⑤ 作成したカード情報（image_url 含む）をフロントに返す
```

### 1.5 Phase 移行の設計方針

`storage.ImageStorage` インターフェースで抽象化することで、
将来的なストレージ先の変更も実装の差し替えのみで完結する。

```go
// backend/internal/storage/storage.go
type ImageStorage interface {
    Upload(ctx context.Context, key string, data []byte, contentType string) (url string, err error)
    Delete(ctx context.Context, key string) error
    GetPublicURL(key string) string
}

// 現在の実装: SupabaseStorage（Phase 1・2 共通）
// 環境変数 STORAGE_BACKEND=supabase で指定
```

---

## 2. リポジトリ構成

Git サブモジュール構成を採用する。

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
├── migrations/            # MySQL マイグレーション SQL
├── Dockerfile
├── docker-compose.yml     # MySQL + Go + Mailpit（ローカル開発用）
├── go.mod
└── .env.example
```

> **Supabase の設定について**
> Supabase は Storage のみ使用するため、専用ディレクトリは不要。
> 接続情報は環境変数（`SUPABASE_URL` / `SUPABASE_SERVICE_KEY`）で管理する。

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

## 3. データベース設計（MySQL）

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
  id            CHAR(36)     NOT NULL DEFAULT (UUID()) COMMENT 'ユーザーID',
  email         VARCHAR(255) NOT NULL                  COMMENT 'メールアドレス',
  password_hash VARCHAR(255) NOT NULL                  COMMENT 'bcrypt ハッシュ',
  display_name  VARCHAR(100) NOT NULL                  COMMENT '表示名',
  created_at    DATETIME     NOT NULL DEFAULT NOW()    COMMENT '登録日時',
  updated_at    DATETIME     NOT NULL DEFAULT NOW()    COMMENT '更新日時',
  PRIMARY KEY (id),
  UNIQUE KEY uq_users_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### ② locations（共有ロケーションマスター）

```sql
CREATE TABLE locations (
  id          CHAR(36)     NOT NULL DEFAULT (UUID())  COMMENT 'ロケーションID',
  name        VARCHAR(100) NOT NULL                   COMMENT 'ロケーション名（例: コンビニ）',
  description TEXT                                    COMMENT '説明文',
  latitude    DOUBLE                                  COMMENT '基準緯度（距離判定用）',
  longitude   DOUBLE                                  COMMENT '基準経度（距離判定用）',
  radius_m    INT          NOT NULL DEFAULT 200       COMMENT '判定半径（メートル）',
  is_default  TINYINT(1)   NOT NULL DEFAULT 1         COMMENT 'デフォルトフラグ',
  created_at  DATETIME     NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 初期シードデータ
INSERT INTO locations (id, name, radius_m) VALUES
  (UUID(), 'コンビニ', 100),
  (UUID(), '病院',     200),
  (UUID(), 'カフェ',   100);
```

#### ③ cards（カードマスター）

`image_url` カラムに Supabase Storage の公開 URL が保存される。

```sql
CREATE TABLE cards (
  id         CHAR(36)     NOT NULL DEFAULT (UUID())   COMMENT 'カードID',
  label      VARCHAR(100) NOT NULL                    COMMENT '表示テキスト（例: おにぎり）',
  image_url  TEXT                                     COMMENT 'Supabase Storage 公開URL',
  emoji      VARCHAR(10)                              COMMENT '絵文字（image_url がない場合）',
  category   VARCHAR(50)                              COMMENT 'カテゴリ（食べ物 / 動詞 等）',
  is_daily   TINYINT(1)   NOT NULL DEFAULT 0          COMMENT '日常カードフラグ',
  created_by CHAR(36)                                 COMMENT 'NULL はシステム共有カード',
  created_at DATETIME     NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id),
  FOREIGN KEY fk_cards_user (created_by)
    REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 日常カードのシード
INSERT INTO cards (id, label, emoji, is_daily) VALUES
  (UUID(), 'こんにちは', '👋', 1),
  (UUID(), 'ありがとう', '🙏', 1),
  (UUID(), 'すみません', '🙇', 1),
  (UUID(), 'はい',       '✅', 1),
  (UUID(), 'いいえ',     '❌', 1);
```

#### ④ location_cards（共有ロケーション × カード）

```sql
CREATE TABLE location_cards (
  id          CHAR(36) NOT NULL DEFAULT (UUID()),
  location_id CHAR(36) NOT NULL COMMENT '共有ロケーションID',
  card_id     CHAR(36) NOT NULL COMMENT 'カードID',
  sort_order  INT      NOT NULL DEFAULT 0,
  created_at  DATETIME NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id),
  UNIQUE KEY uq_loc_card (location_id, card_id),
  FOREIGN KEY fk_lc_location (location_id)
    REFERENCES locations(id) ON DELETE CASCADE,
  FOREIGN KEY fk_lc_card (card_id)
    REFERENCES cards(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### ⑤ user_locations（ユーザー独自ロケーション）

```sql
CREATE TABLE user_locations (
  id         CHAR(36)     NOT NULL DEFAULT (UUID()),
  user_id    CHAR(36)     NOT NULL COMMENT '所有ユーザーID',
  name       VARCHAR(100) NOT NULL COMMENT '場所名（例: ピアノ教室）',
  latitude   DOUBLE       NOT NULL COMMENT '緯度',
  longitude  DOUBLE       NOT NULL COMMENT '経度',
  radius_m   INT          NOT NULL DEFAULT 100 COMMENT '判定半径（メートル）',
  created_at DATETIME     NOT NULL DEFAULT NOW(),
  updated_at DATETIME     NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id),
  FOREIGN KEY fk_ul_user (user_id)
    REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### ⑥ user_location_cards（ユーザーロケーション × カード）

```sql
CREATE TABLE user_location_cards (
  id               CHAR(36) NOT NULL DEFAULT (UUID()),
  user_location_id CHAR(36) NOT NULL COMMENT 'ユーザーロケーションID',
  card_id          CHAR(36) NOT NULL COMMENT 'カードID',
  sort_order       INT      NOT NULL DEFAULT 0,
  created_at       DATETIME NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id),
  UNIQUE KEY uq_ulc (user_location_id, card_id),
  FOREIGN KEY fk_ulc_location (user_location_id)
    REFERENCES user_locations(id) ON DELETE CASCADE,
  FOREIGN KEY fk_ulc_card (card_id)
    REFERENCES cards(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.3 インデックス

```sql
-- ロケーション距離判定（緯度経度での概算フィルタ）
CREATE INDEX idx_locations_latlng       ON locations(latitude, longitude);
CREATE INDEX idx_user_locations_user    ON user_locations(user_id);
CREATE INDEX idx_user_locations_latlng  ON user_locations(latitude, longitude);

-- カード取得の高速化
CREATE INDEX idx_location_cards_loc     ON location_cards(location_id, sort_order);
CREATE INDEX idx_ulc_ul                 ON user_location_cards(user_location_id, sort_order);
CREATE INDEX idx_cards_is_daily         ON cards(is_daily);
CREATE INDEX idx_cards_created_by       ON cards(created_by);
```

---

## 4. API 設計（Go / Gin）

### 4.1 共通仕様

| 項目 | 内容 |
|---|---|
| Base URL | `/api/v1` |
| 認証ヘッダー | `Authorization: Bearer <access_token>` |
| Content-Type | `application/json`（カード作成のみ `multipart/form-data`） |
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
| GET    | `/locations/nearby`         | 不要 | 現在地(lat,lng)から半径内のロケーションを距離順で返す |
| GET    | `/locations`                | 不要 | 共有ロケーション一覧 |
| GET    | `/locations/:id/cards`      | 不要 | 共有ロケーションのカード一覧 |
| GET    | `/user/locations`           | 必要 | ユーザー独自ロケーション一覧 |
| POST   | `/user/locations`           | 必要 | 独自ロケーション追加 |
| PUT    | `/user/locations/:id`       | 必要 | 独自ロケーション更新 |
| DELETE | `/user/locations/:id`       | 必要 | 独自ロケーション削除 |
| GET    | `/user/locations/:id/cards` | 必要 | ユーザーロケーションのカード一覧 |

**GET /locations/nearby クエリパラメータ例**

```
GET /api/v1/locations/nearby?lat=33.2500&lng=131.6100&radius_m=500
```

```json
[
  {
    "id":          "char36-uuid",
    "name":        "コンビニ",
    "type":        "shared",
    "distance_m":  42,
    "cards_count": 12
  }
]
```

> **Haversine 公式について**
> Haversine 公式とは、球面上の 2 地点間の距離を緯度・経度から計算する公式。
> `GET /locations/nearby` では DB クエリで `±0.01度` の概算範囲フィルタをかけて候補を絞り込み、
> Go のサービス層で Haversine 公式による正確な距離計算を行い、
> `radius_m` 以内のロケーションのみ距離昇順で返す。

### 4.4 カード API（`/api/v1/cards`）

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| GET    | `/cards/daily`                       | 不要 | 日常カード一覧（is_daily=1） |
| POST   | `/user/cards`                        | 必要 | カード作成（画像アップロード＋DB保存を1リクエストで完結） |
| POST   | `/user/locations/:id/cards`          | 必要 | ロケーションにカードを追加（共有カードも可） |
| DELETE | `/user/locations/:id/cards/:card_id` | 必要 | ロケーションからカードを削除 |
| PUT    | `/user/locations/:id/cards/reorder`  | 必要 | カード表示順の一括更新 |

**POST /user/cards（案A・画像＋DB保存を1リクエスト）**

```
POST /api/v1/user/cards
Content-Type: multipart/form-data

フィールド:
  file:     <画像バイナリ>（任意）
  label:    "おにぎり"
  emoji:    "🍙"（file がない場合に使用）
  category: "食べ物"
```

```
Go 内部処理:
  1. file を取り出し Supabase Storage にアップロード
  2. 公開 URL を取得
  3. cards テーブルに INSERT（image_url に URL を保存）
  4. 作成したカード情報をレスポンスで返す
```

```json
// レスポンス
{
  "id":        "char36-uuid",
  "label":     "おにぎり",
  "image_url": "https://xxxx.supabase.co/storage/v1/object/public/cards/uuid.jpg",
  "emoji":     null,
  "category":  "食べ物",
  "is_daily":  false,
  "created_at": "2025-01-01T00:00:00Z"
}
```

### 4.5 AI レコメンド API

| メソッド | パス | 認証 | 説明 |
|---|---|---|---|
| POST | `/ai/recommend` | 不要 | 選択単語配列 → 自然な文章候補を返す（Gemini 2.5 Flash） |

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
> AI 生成中も元の単語列をそのまま発音ボタンで発話できる設計を維持する。

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
│   │   └── db.go                     # MySQL 接続（go-sql-driver/mysql）
│   │
│   ├── handler/                      # Gin ハンドラ層（薄く保つ）
│   │   ├── auth.go                   # POST /auth/*
│   │   ├── location.go               # GET /locations/*
│   │   ├── card.go                   # GET,POST /cards/*
│   │   ├── user_location.go          # CRUD /user/locations/*
│   │   └── ai.go                     # POST /ai/recommend
│   │
│   ├── middleware/
│   │   ├── auth.go                   # JWT 検証ミドルウェア
│   │   └── cors.go                   # CORS 設定
│   │
│   ├── service/                      # ビジネスロジック層
│   │   ├── auth_service.go           # JWT 発行・検証・bcrypt
│   │   ├── location_service.go       # Haversine 距離計算
│   │   ├── card_service.go           # カード CRUD・画像アップロード呼び出し
│   │   └── ai_service.go             # Gemini 2.5 Flash API 呼び出し
│   │
│   ├── storage/
│   │   ├── storage.go                # ImageStorage インターフェース定義
│   │   └── supabase.go               # SupabaseStorage 実装
│   │                                 # Upload() → Supabase Storage に保存 → 公開URL返却
│   │                                 # 返却した URL を card_service が cards.image_url に保存
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
├── migrations/                       # MySQL マイグレーション SQL
│   ├── 001_create_users.sql
│   ├── 002_create_locations.sql
│   ├── 003_create_cards.sql
│   ├── 004_create_location_cards.sql
│   ├── 005_create_user_locations.sql
│   ├── 006_create_user_location_cards.sql
│   └── 007_seed_initial_data.sql     # コンビニ/病院/カフェ + 日常カード
│
├── Dockerfile
├── docker-compose.yml                # MySQL + Go + Mailpit（ローカル開発用）
├── go.mod
├── go.sum
└── .env.example
```

### 5.2 tsu_hackF（フロントエンド）

```
tsu_hackF/
├── src/
│   ├── app/                              # App Router
│   │   ├── layout.tsx
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
│   │   │   ├── CardGrid.tsx
│   │   │   ├── CardItem.tsx
│   │   │   ├── SelectedList.tsx          # 選択リスト（ストック箱）
│   │   │   ├── AiSuggestion.tsx          # AI レコメンド表示領域
│   │   │   └── SpeakButton.tsx           # 発音ボタン（Web Speech API）
│   │   ├── location/
│   │   │   ├── LocationFinder.tsx        # GPS 取得 → 近隣ロケーション提案
│   │   │   └── LocationChanger.tsx       # ロケーション変更ボタン
│   │   ├── edit/
│   │   │   ├── CardEditor.tsx            # カード追加フォーム
│   │   │   └── ImageUploader.tsx         # 画像選択・プレビュー・送信
│   │   └── ui/
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
│   │   └── geo.ts                        # 距離計算ユーティリティ
│   │
│   ├── store/
│   │   └── boardStore.ts                 # Zustand: 選択カード・AI 候補状態
│   │                                     # ※ "use client" コンポーネント内でのみ使用
│   └── types/
│       ├── location.ts
│       ├── card.ts
│       └── auth.ts
│
├── public/icons/
├── next.config.ts
├── tailwind.config.ts
└── package.json
```

---

## 6. ローカル開発環境

### 6.1 起動方法

```bash
# 1. MySQL + Go + Mailpit を Docker で起動（tsu_hackB 内）
docker compose up -d

# 2. マイグレーション実行
docker compose exec backend go run ./cmd/migrate

# 3. Next.js をローカルで起動（tsu_hackF 内）
npm run dev
```

### 6.2 docker-compose.yml（tsu_hackB）

Next.js は含めない。MySQL・Go・Mailpit の 3 サービスのみ。

```yaml
services:
  db:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: aac_dev
      MYSQL_USER: aac
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - mysqldata:/var/lib/mysql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

  backend:
    build: .
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      - db

  mailpit:
    image: axllent/mailpit
    ports:
      - "8025:8025"   # Web UI（メール確認）
      - "1025:1025"   # SMTP（バックエンドからの送信先）

volumes:
  mysqldata:
```

### 6.3 Mailpit について

Mailpit は追加設定不要のローカル用メールキャッチャー。
バックエンドの SMTP 送信先を `localhost:1025` に向けるだけで、
送信されたメールをブラウザ（`http://localhost:8025`）で確認できる。
アカウント登録・外部サービス契約は一切不要。

---

## 7. 環境変数

### 7.1 バックエンド（`backend/.env`）

```env
# サーバー
PORT=8080
ENV=development   # development | production

# MySQL
# Phase 1: Docker MySQL
# Phase 2: Render MySQL 接続文字列に変更
DATABASE_URL=aac:password@tcp(localhost:3306)/aac_dev?charset=utf8mb4&parseTime=True

# JWT
JWT_SECRET=your-secret-key-min-32chars
JWT_ACCESS_EXPIRES_MIN=15
JWT_REFRESH_EXPIRES_DAYS=7

# Supabase Storage（Phase 1・2 共通）
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
- Supabase Storage のバケットポリシーで認証済みユーザーのみアップロード可能に設定

### 8.3 API 共通

- 全エンドポイントに CORS 設定（許可オリジンを環境変数で管理）
- Gin の Rate Limiting ミドルウェアで連続リクエスト制限
- DB クエリはプリペアドステートメントのみ使用（SQL インジェクション防止）
- HTTPS 経由のみで `Authorization` ヘッダーを送信（Phase 2: Render + Vercel はデフォルト HTTPS）
