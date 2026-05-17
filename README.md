# 営業日報システム

営業担当者の日次訪問報告とマネージャーフィードバックを管理する Web アプリケーション。

## 機能

- 営業が当日訪問した顧客と訪問内容を日報として記録（1日に複数件追加可能）
- 課題・相談（Problem）と翌日計画（Plan）の記載
- マネージャーによる Problem / Plan へのコメント
- 顧客マスタ・ユーザー管理（マネージャーのみ）

## 技術スタック

| カテゴリ       | 技術                                 |
| -------------- | ------------------------------------ |
| フレームワーク | Next.js 14+ (TypeScript, App Router) |
| DB / ORM       | PostgreSQL + Prisma                  |
| 認証           | NextAuth.js                          |
| ホスティング   | Google Cloud Run                     |

## セットアップ

```bash
# 依存関係のインストール
npm install

# 環境変数の設定
cp .env.example .env.local

# DBマイグレーション
npx prisma migrate dev

# 開発サーバーの起動
npm run dev
```

## ドキュメント

| ドキュメント                                    | 説明                                             |
| ----------------------------------------------- | ------------------------------------------------ |
| [DESIGN.md](docs/superpowers/specs/DESIGN.md)   | システム設計・ER図・テーブル定義・アーキテクチャ |
| [SCREENS.md](docs/superpowers/specs/SCREENS.md) | 画面定義・画面遷移図                             |
| [API.md](docs/superpowers/specs/API.md)         | API仕様                                          |
| [TESTS.md](docs/superpowers/specs/TESTS.md)     | テスト仕様                                       |
