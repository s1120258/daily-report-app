# 営業日報システム

営業担当者の日次訪問報告とマネージャーフィードバックを管理する Web アプリケーション。

## 仕様

@docs/superpowers/specs/DESIGN.md

タスクに応じて作業開始前に該当ファイルを読むこと:

- 画面・UI の実装時 → `docs/superpowers/specs/SCREENS.md`
- API Route Handler の実装時 → `docs/superpowers/specs/API.md`
- テストの作成時 → `docs/superpowers/specs/TESTS.md`

## 技術スタック

- **Framework**: Next.js (TypeScript, App Router)
- **DB / ORM**: PostgreSQL + Prisma
- **認証**: NextAuth.js
- **ホスティング**: Google Cloud Run
- **CI/CD**: GitHub Actions（typecheck / lint / format / test）

## コーディング規約

- コンポーネントは `app/` 配下に App Router の規約に従って配置する
- API は `app/api/` の Route Handlers として実装する
- DB アクセスは必ず Prisma 経由で行う（生 SQL 禁止）
- 権限チェックは middleware と Route Handler の両方で行う（二重防御）
- `sales` ロールは自分のリソースにのみアクセスできる。サーバーサイドで必ず検証する
