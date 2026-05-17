# 営業日報システム API仕様書

作成日: 2026-05-17

---

## 1. 共通仕様

### ベースURL

```
/api
```

### 認証

NextAuth.js のセッションクッキーによる認証。全エンドポイント（`/api/auth/*` を除く）で有効なセッションが必要。未認証の場合は `401` を返す。

### リクエストヘッダー

| ヘッダー | 値 |
| --- | --- |
| Content-Type | application/json |

### レスポンス形式

成功・エラーともに JSON を返す。

**成功レスポンス例:**
```json
{
  "data": { ... }
}
```

**エラーレスポンス例:**
```json
{
  "error": "エラーメッセージ"
}
```

### HTTPステータスコード

| コード | 意味 |
| --- | --- |
| 200 | 成功 |
| 201 | 作成成功 |
| 400 | リクエスト不正（バリデーションエラー） |
| 401 | 未認証 |
| 403 | 権限なし |
| 404 | リソースが存在しない |
| 409 | 競合（同一日の日報が既に存在するなど） |
| 500 | サーバーエラー |

### ロール権限

| ロール | 説明 |
| --- | --- |
| `sales` | 自分のリソースのみ作成・編集可 |
| `manager` | 全ユーザーのリソースを閲覧可。顧客・ユーザー管理が可能 |

---

## 2. 認証 API

NextAuth.js が自動生成するエンドポイント。

| メソッド | パス | 説明 |
| --- | --- | --- |
| GET / POST | `/api/auth/[...nextauth]` | ログイン・ログアウト・セッション取得 |

### セッション情報

NextAuth.js のセッションに含まれるユーザー情報:

```json
{
  "user": {
    "id": 1,
    "name": "山田 太郎",
    "email": "yamada@example.com",
    "role": "sales"
  }
}
```

---

## 3. 日報 API

### 3.1 日報一覧取得

```
GET /api/daily-reports
```

**権限:** 全ユーザー（`sales` は自分の日報のみ返却。`manager` は全員分）

**クエリパラメータ:**

| パラメータ | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| user_id | number | 任意 | 担当者で絞り込み（`manager` のみ使用可） |
| from | string (YYYY-MM-DD) | 任意 | 期間開始日 |
| to | string (YYYY-MM-DD) | 任意 | 期間終了日 |

**レスポンス `200`:**

```json
{
  "data": [
    {
      "id": 1,
      "reportDate": "2026-05-17",
      "user": {
        "id": 2,
        "name": "鈴木 一郎"
      },
      "visitCount": 3,
      "hasProblemComment": true,
      "hasPlanComment": false
    }
  ]
}
```

---

### 3.2 日報作成

```
POST /api/daily-reports
```

**権限:** `sales` のみ

**リクエストボディ:**

```json
{
  "reportDate": "2026-05-17",
  "problemText": "A社での予算交渉が難航している。上長に相談したい。",
  "planText": "B社に提案書を持参して訪問する。",
  "visitRecords": [
    {
      "customerId": 10,
      "visitedAt": "2026-05-17T10:00:00+09:00",
      "content": "新製品の紹介。次回デモを希望。"
    },
    {
      "customerId": 15,
      "visitedAt": "2026-05-17T14:30:00+09:00",
      "content": "契約更新の確認。来週中に書類を送付する。"
    }
  ]
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| reportDate | string (YYYY-MM-DD) | 必須 | 日報対象日 |
| problemText | string | 任意 | 課題・相談 |
| planText | string | 任意 | 明日やること |
| visitRecords | array | 任意 | 訪問記録（0件以上） |
| visitRecords[].customerId | number | 必須 | 顧客マスタの ID |
| visitRecords[].visitedAt | string (ISO 8601) | 必須 | 訪問日時 |
| visitRecords[].content | string | 必須 | 訪問内容 |

**レスポンス `201`:**

```json
{
  "data": {
    "id": 42
  }
}
```

**エラー `409`:** 同一ユーザー・同一日の日報が既に存在する場合

```json
{
  "error": "この日付の日報は既に作成されています。"
}
```

---

### 3.3 日報詳細取得

```
GET /api/daily-reports/[id]
```

**権限:** `sales` は自分の日報のみ。`manager` は全員分。

**レスポンス `200`:**

```json
{
  "data": {
    "id": 42,
    "reportDate": "2026-05-17",
    "user": {
      "id": 2,
      "name": "鈴木 一郎"
    },
    "problemText": "A社での予算交渉が難航している。上長に相談したい。",
    "planText": "B社に提案書を持参して訪問する。",
    "problemComment": {
      "text": "来週一緒に訪問しましょう。",
      "commentedBy": { "id": 1, "name": "田中 部長" },
      "commentedAt": "2026-05-17T18:30:00+09:00"
    },
    "planComment": null,
    "visitRecords": [
      {
        "id": 101,
        "customer": {
          "id": 10,
          "name": "株式会社A"
        },
        "visitedAt": "2026-05-17T10:00:00+09:00",
        "content": "新製品の紹介。次回デモを希望。"
      }
    ],
    "createdAt": "2026-05-17T17:00:00+09:00",
    "updatedAt": "2026-05-17T17:30:00+09:00"
  }
}
```

`problemComment` / `planComment` はコメント未記入の場合 `null`。

---

### 3.4 日報更新

```
PUT /api/daily-reports/[id]
```

**権限:** `sales` かつ自分の日報のみ

**リクエストボディ:**

```json
{
  "problemText": "A社の件、追加情報あり。",
  "planText": "B社訪問に加え、C社にも連絡する。",
  "visitRecords": [
    {
      "id": 101,
      "customerId": 10,
      "visitedAt": "2026-05-17T10:00:00+09:00",
      "content": "新製品の紹介。次回デモを希望。"
    },
    {
      "customerId": 20,
      "visitedAt": "2026-05-17T15:00:00+09:00",
      "content": "初回挨拶。担当者と名刺交換。"
    }
  ]
}
```

> `visitRecords` は全件送信（差分ではなく洗い替え）。`id` あり → 更新、`id` なし → 新規追加。送信されなかった既存レコードは削除。

**レスポンス `200`:**

```json
{
  "data": {
    "id": 42
  }
}
```

---

### 3.5 Problem コメント更新

```
PATCH /api/daily-reports/[id]/problem-comment
```

**権限:** `manager` のみ

**リクエストボディ:**

```json
{
  "text": "来週一緒に訪問しましょう。"
}
```

**レスポンス `200`:**

```json
{
  "data": {
    "text": "来週一緒に訪問しましょう。",
    "commentedBy": { "id": 1, "name": "田中 部長" },
    "commentedAt": "2026-05-17T18:30:00+09:00"
  }
}
```

---

### 3.6 Plan コメント更新

```
PATCH /api/daily-reports/[id]/plan-comment
```

**権限:** `manager` のみ

リクエスト・レスポンス形式は [3.5 Problem コメント更新](#35-problem-コメント更新) と同様。

---

## 4. 顧客 API

### 4.1 顧客一覧取得

```
GET /api/customers
```

**権限:** 全ユーザー

**レスポンス `200`:**

```json
{
  "data": [
    {
      "id": 10,
      "name": "株式会社A",
      "contactName": "佐藤 花子",
      "phone": "03-1234-5678",
      "email": "sato@example.com"
    }
  ]
}
```

---

### 4.2 顧客登録

```
POST /api/customers
```

**権限:** `manager` のみ

**リクエストボディ:**

```json
{
  "name": "株式会社B",
  "contactName": "高橋 次郎",
  "phone": "06-9876-5432",
  "email": "takahashi@example-b.com"
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| name | string | 必須 | 顧客名・会社名 |
| contactName | string | 任意 | 担当者名 |
| phone | string | 任意 | 電話番号 |
| email | string | 任意 | メールアドレス |

**レスポンス `201`:**

```json
{
  "data": {
    "id": 11
  }
}
```

---

### 4.3 顧客更新

```
PUT /api/customers/[id]
```

**権限:** `manager` のみ

リクエストボディは [4.2 顧客登録](#42-顧客登録) と同様。

**レスポンス `200`:**

```json
{
  "data": {
    "id": 11
  }
}
```

---

## 5. ユーザー API

### 5.1 ユーザー一覧取得

```
GET /api/users
```

**権限:** `manager` のみ

**レスポンス `200`:**

```json
{
  "data": [
    {
      "id": 2,
      "name": "鈴木 一郎",
      "email": "suzuki@example.com",
      "role": "sales"
    }
  ]
}
```

---

### 5.2 ユーザー登録

```
POST /api/users
```

**権限:** `manager` のみ

**リクエストボディ:**

```json
{
  "name": "山本 三郎",
  "email": "yamamoto@example.com",
  "password": "初期パスワード",
  "role": "sales"
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| name | string | 必須 | 氏名 |
| email | string | 必須 | メールアドレス（ログインID） |
| password | string | 必須 | 初期パスワード |
| role | string | 必須 | `sales` / `manager` |

**レスポンス `201`:**

```json
{
  "data": {
    "id": 5
  }
}
```

**エラー `409`:** メールアドレスが既に登録済みの場合

```json
{
  "error": "このメールアドレスは既に使用されています。"
}
```

---

### 5.3 ユーザー更新

```
PUT /api/users/[id]
```

**権限:** `manager` のみ

**リクエストボディ:**

```json
{
  "name": "山本 三郎",
  "email": "yamamoto@example.com",
  "password": "",
  "role": "manager"
}
```

> `password` が空文字の場合はパスワードを変更しない。

**レスポンス `200`:**

```json
{
  "data": {
    "id": 5
  }
}
```

---

## 6. エンドポイント一覧

| メソッド | パス | 説明 | 権限 |
| --- | --- | --- | --- |
| GET | `/api/daily-reports` | 日報一覧取得 | 全員 |
| POST | `/api/daily-reports` | 日報作成 | sales |
| GET | `/api/daily-reports/[id]` | 日報詳細取得 | 全員 |
| PUT | `/api/daily-reports/[id]` | 日報更新 | sales（自分のみ） |
| PATCH | `/api/daily-reports/[id]/problem-comment` | Problem コメント更新 | manager |
| PATCH | `/api/daily-reports/[id]/plan-comment` | Plan コメント更新 | manager |
| GET | `/api/customers` | 顧客一覧取得 | 全員 |
| POST | `/api/customers` | 顧客登録 | manager |
| PUT | `/api/customers/[id]` | 顧客更新 | manager |
| GET | `/api/users` | ユーザー一覧取得 | manager |
| POST | `/api/users` | ユーザー登録 | manager |
| PUT | `/api/users/[id]` | ユーザー更新 | manager |
