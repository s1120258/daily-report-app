# 営業日報システム

営業担当者の日次訪問報告とマネージャーフィードバックを管理する Web アプリケーション。

## 仕様

@docs/superpowers/specs/DESIGN.md

その他の仕様書:
- 画面定義: `docs/superpowers/specs/SCREENS.md`
- API仕様: `docs/superpowers/specs/API.md`
- テスト仕様: `docs/superpowers/specs/TESTS.md`

## 技術スタック

- **Framework**: Next.js (TypeScript, App Router)
- **DB / ORM**: PostgreSQL + Prisma
- **認証**: NextAuth.js
- **ホスティング**: Vercel

## コーディング規約

- コンポーネントは `app/` 配下に App Router の規約に従って配置する
- API は `app/api/` の Route Handlers として実装する
- DB アクセスは必ず Prisma 経由で行う（生 SQL 禁止）
- 権限チェックは middleware と Route Handler の両方で行う（二重防御）
- `sales` ロールは自分のリソースにのみアクセスできる。サーバーサイドで必ず検証する
