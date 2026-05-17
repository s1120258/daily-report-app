# 営業日報システム API仕様書

## エンドポイント一覧

| メソッド | パス | 説明 | 権限 |
| --- | --- | --- | --- |
| GET | `/api/daily-reports` | 日報一覧 | 全員 |
| POST | `/api/daily-reports` | 日報作成 | sales |
| GET | `/api/daily-reports/[id]` | 日報詳細 | 全員 |
| PUT | `/api/daily-reports/[id]` | 日報更新 | sales（自分のみ） |
| PATCH | `/api/daily-reports/[id]/problem-comment` | Problem コメント更新 | manager |
| PATCH | `/api/daily-reports/[id]/plan-comment` | Plan コメント更新 | manager |
| GET | `/api/customers` | 顧客一覧 | 全員 |
| POST | `/api/customers` | 顧客登録 | manager |
| PUT | `/api/customers/[id]` | 顧客更新 | manager |
| GET | `/api/users` | ユーザー一覧 | manager |
| POST | `/api/users` | ユーザー登録 | manager |
| PUT | `/api/users/[id]` | ユーザー更新 | manager |

## 共通仕様

- **ベースURL**: `/api`
- **認証**: NextAuth.js セッションクッキー。未認証は `401`。権限不足は `403`。
- **Content-Type**: `application/json`
- **レスポンス形式**: 成功 `{"data": ...}`、エラー `{"error": "メッセージ"}`

| コード | 意味 |
| --- | --- |
| 200 / 201 | 成功 / 作成成功 |
| 400 | バリデーションエラー |
| 401 / 403 | 未認証 / 権限なし |
| 404 / 409 | リソースなし / 競合 |

## 認証 API

NextAuth.js が自動生成。`GET/POST /api/auth/[...nextauth]`（ログイン・ログアウト・セッション取得）

セッションに含まれるユーザー情報:
```json
{ "user": { "id": 1, "name": "山田 太郎", "email": "yamada@example.com", "role": "sales" } }
```

## 日報 API

### `GET /api/daily-reports`

`sales` は自分の日報のみ返却。`manager` は全員分。

| クエリ | 型 | 説明 |
| --- | --- | --- |
| user_id | number | 担当者絞り込み（manager のみ有効） |
| from / to | YYYY-MM-DD | 期間絞り込み |

```json
{ "data": [{ "id": 1, "reportDate": "2026-05-17", "user": { "id": 2, "name": "鈴木 一郎" }, "visitCount": 3, "hasProblemComment": true, "hasPlanComment": false }] }
```

### `POST /api/daily-reports`

権限: `sales` のみ。同一日の重複は `409`。

| フィールド | 型 | 必須 |
| --- | --- | --- |
| reportDate | YYYY-MM-DD | 必須 |
| problemText / planText | string | 任意 |
| visitRecords[].customerId | number | 必須 |
| visitRecords[].visitedAt | ISO 8601 | 必須 |
| visitRecords[].content | string | 必須 |

レスポンス `201`: `{"data": {"id": 42}}`

### `GET /api/daily-reports/[id]`

`sales` は自分の日報のみ（他人は `403`）。

```json
{
  "data": {
    "id": 42,
    "reportDate": "2026-05-17",
    "user": { "id": 2, "name": "鈴木 一郎" },
    "problemText": "A社での予算交渉が難航している。",
    "planText": "B社に提案書を持参して訪問する。",
    "problemComment": {
      "text": "来週一緒に訪問しましょう。",
      "commentedBy": { "id": 1, "name": "田中 部長" },
      "commentedAt": "2026-05-17T18:30:00+09:00"
    },
    "planComment": null,
    "visitRecords": [
      { "id": 101, "customer": { "id": 10, "name": "株式会社A" }, "visitedAt": "2026-05-17T10:00:00+09:00", "content": "新製品の紹介。次回デモを希望。" }
    ],
    "createdAt": "2026-05-17T17:00:00+09:00",
    "updatedAt": "2026-05-17T17:30:00+09:00"
  }
}
```

`problemComment` / `planComment` はコメント未記入の場合 `null`。

### `PUT /api/daily-reports/[id]`

権限: `sales` かつ自分の日報のみ。

`visitRecords` は洗い替え方式。`id` あり → 更新、`id` なし → 追加、送信されなかった既存レコード → 削除。

| フィールド | 型 | 必須 |
| --- | --- | --- |
| problemText / planText | string | 任意 |
| visitRecords[].id | number | 任意（既存行の場合） |
| visitRecords[].customerId | number | 必須 |
| visitRecords[].visitedAt | ISO 8601 | 必須 |
| visitRecords[].content | string | 必須 |

レスポンス `200`: `{"data": {"id": 42}}`

### `PATCH /api/daily-reports/[id]/problem-comment`

権限: `manager` のみ。既存コメントは上書き。

リクエスト: `{"text": "来週一緒に訪問しましょう。"}`

```json
{ "data": { "text": "来週一緒に訪問しましょう。", "commentedBy": { "id": 1, "name": "田中 部長" }, "commentedAt": "2026-05-17T18:30:00+09:00" } }
```

### `PATCH /api/daily-reports/[id]/plan-comment`

権限: `manager` のみ。リクエスト・レスポンス形式は `problem-comment` と同様。

## 顧客 API

### `GET /api/customers`

レスポンス `200`:
```json
{ "data": [{ "id": 10, "name": "株式会社A", "contactName": "佐藤 花子", "phone": "03-1234-5678", "email": "sato@example.com" }] }
```

### `POST /api/customers` / `PUT /api/customers/[id]`

権限: `manager` のみ。

| フィールド | 型 | 必須 |
| --- | --- | --- |
| name | string | 必須 |
| contactName / phone / email | string | 任意 |

レスポンス: POST `201` / PUT `200` → `{"data": {"id": 11}}`

## ユーザー API

### `GET /api/users`

権限: `manager` のみ。

レスポンス `200`:
```json
{ "data": [{ "id": 2, "name": "鈴木 一郎", "email": "suzuki@example.com", "role": "sales" }] }
```

### `POST /api/users` / `PUT /api/users/[id]`

権限: `manager` のみ。メール重複は `409`。

| フィールド | 型 | 必須 | 備考 |
| --- | --- | --- | --- |
| name | string | 必須 | |
| email | string | 必須 | ログインID |
| password | string | POST は必須 | PUT で空文字 → 変更なし |
| role | string | 必須 | `sales` / `manager` |

レスポンス: POST `201` / PUT `200` → `{"data": {"id": 5}}`
