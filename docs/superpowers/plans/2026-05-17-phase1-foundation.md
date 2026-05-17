# Phase 1: Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Next.js プロジェクトのセットアップ、DB スキーマ、認証、CI/CD ガードレールを整える。

**Architecture:** Next.js App Router + PostgreSQL (Prisma) + NextAuth.js + GitHub Actions CI + husky ガードレール + Google Cloud Run 用 Dockerfile。

**Tech Stack:** Next.js 14, TypeScript, Prisma, NextAuth.js, bcryptjs, Jest, Vitest, Playwright, Prettier, ESLint, husky, lint-staged

**Phases:**
- [Phase 1 (本ファイル)](2026-05-17-phase1-foundation.md) — Foundation
- [Phase 2](2026-05-17-phase2-api.md) — API Layer
- [Phase 3](2026-05-17-phase3-ui.md) — UI Layer
- [Phase 4](2026-05-17-phase4-tests.md) — Tests & Quality

---

## Task 1: プロジェクトセットアップ

**Files:**
- Create: `package.json`（Next.js プロジェクト）
- Create: `.env.local`, `.env.test`
- Create: `jest.config.ts`, `vitest.config.ts`
- Create: `.prettierrc`, `.prettierignore`
- Create: `next.config.ts`

- [ ] **Step 1: Next.js プロジェクトを作成**

```bash
npx create-next-app@latest . \
  --typescript \
  --app \
  --no-src-dir \
  --import-alias "@/*" \
  --no-tailwind
```

- [ ] **Step 2: 依存パッケージをインストール**

```bash
npm install next-auth @auth/prisma-adapter prisma @prisma/client bcryptjs
npm install --save-dev @types/bcryptjs \
  jest @types/jest ts-jest jest-environment-node \
  vitest @vitest/coverage-v8 \
  @playwright/test \
  prettier eslint-config-prettier \
  husky lint-staged
```

- [ ] **Step 3: Prisma を初期化**

```bash
npx prisma init
```

- [ ] **Step 4: 環境変数ファイルを作成**

`.env.local`:
```
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report"
NEXTAUTH_SECRET="dev-secret-change-in-production"
NEXTAUTH_URL="http://localhost:3000"
```

`.env.test`:
```
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test"
NEXTAUTH_SECRET="test-secret"
NEXTAUTH_URL="http://localhost:3000"
```

- [ ] **Step 5: `.gitignore` に秘密情報が含まれていることを確認**

```bash
grep ".env" .gitignore
```

Expected: `.env*.local` または `.env.local` がリストされている。`.env.test` は機密値を含む場合は追加する。

- [ ] **Step 6: `next.config.ts` に standalone 出力を設定（Cloud Run 用）**

```typescript
// next.config.ts
import type { NextConfig } from "next"

const nextConfig: NextConfig = {
  output: "standalone",
}

export default nextConfig
```

- [ ] **Step 7: `jest.config.ts` を作成（API 統合テスト用）**

```typescript
// jest.config.ts
import type { Config } from "jest"
import nextJest from "next/jest"

const createJestConfig = nextJest({ dir: "./" })

const config: Config = {
  testEnvironment: "node",
  setupFilesAfterFramework: ["<rootDir>/__tests__/setup.ts"],
  moduleNameMapper: { "^@/(.*)$": "<rootDir>/$1" },
  testMatch: ["**/__tests__/api/**/*.test.ts"],
}

export default createJestConfig(config)
```

- [ ] **Step 8: `vitest.config.ts` を作成（単体テスト用）**

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config"
import path from "path"

export default defineConfig({
  test: {
    environment: "node",
    include: ["__tests__/unit/**/*.test.ts"],
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, ".") },
  },
})
```

- [ ] **Step 9: Prettier を設定**

`.prettierrc`:
```json
{
  "semi": false,
  "singleQuote": false,
  "trailingComma": "es5",
  "printWidth": 100
}
```

`.prettierignore`:
```
.next/
node_modules/
prisma/migrations/
```

`.eslintrc.json` に `prettier` を追加（Next.js が生成したファイルを編集）:
```json
{
  "extends": ["next/core-web-vitals", "next/typescript", "prettier"]
}
```

- [ ] **Step 10: `package.json` スクリプトを設定**

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "typecheck": "tsc --noEmit",
  "lint": "next lint",
  "format": "prettier --write .",
  "test": "jest --no-coverage",
  "test:unit": "vitest run",
  "test:e2e": "playwright test"
}
```

- [ ] **Step 11: コミット**

```bash
git add -A
git commit -m "chore: initial Next.js project setup with testing and formatting tools"
```

---

## Task 2: Prisma スキーマ & マイグレーション

**Files:**
- Create: `prisma/schema.prisma`
- Create: `prisma/seed.ts`
- Create: `lib/prisma.ts`
- Create: `lib/utils.ts`（BigInt シリアライズ・日付フォーマット）

- [ ] **Step 1: `prisma/schema.prisma` を定義**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  sales
  manager
}

model User {
  id              BigInt        @id @default(autoincrement())
  name            String
  email           String        @unique
  passwordHash    String
  role            Role
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  dailyReports    DailyReport[] @relation("ReportAuthor")
  problemComments DailyReport[] @relation("ProblemCommenter")
  planComments    DailyReport[] @relation("PlanCommenter")
}

model Customer {
  id           BigInt        @id @default(autoincrement())
  name         String
  contactName  String?
  phone        String?
  email        String?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
  visitRecords VisitRecord[]
}

model DailyReport {
  id                 BigInt        @id @default(autoincrement())
  userId             BigInt
  reportDate         DateTime      @db.Date
  problemText        String?
  planText           String?
  problemCommentText String?
  problemCommentBy   BigInt?
  problemCommentedAt DateTime?
  planCommentText    String?
  planCommentBy      BigInt?
  planCommentedAt    DateTime?
  createdAt          DateTime      @default(now())
  updatedAt          DateTime      @updatedAt

  user             User          @relation("ReportAuthor", fields: [userId], references: [id])
  problemCommenter User?         @relation("ProblemCommenter", fields: [problemCommentBy], references: [id])
  planCommenter    User?         @relation("PlanCommenter", fields: [planCommentBy], references: [id])
  visitRecords     VisitRecord[]

  @@unique([userId, reportDate])
}

model VisitRecord {
  id            BigInt      @id @default(autoincrement())
  dailyReportId BigInt
  customerId    BigInt
  visitedAt     DateTime
  content       String
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  dailyReport DailyReport @relation(fields: [dailyReportId], references: [id], onDelete: Cascade)
  customer    Customer    @relation(fields: [customerId], references: [id])
}
```

- [ ] **Step 2: `lib/prisma.ts` を作成**

```typescript
// lib/prisma.ts
import { PrismaClient } from "@prisma/client"

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({ log: process.env.NODE_ENV === "development" ? ["query"] : [] })

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma
```

- [ ] **Step 3: `lib/utils.ts` を作成**

```typescript
// lib/utils.ts
export function serializeId(id: bigint): string {
  return String(id)
}

export function formatDate(date: Date): string {
  return date.toISOString().slice(0, 10)
}
```

- [ ] **Step 4: マイグレーションを実行**

```bash
npx prisma migrate dev --name init
```

Expected: `✔ Generated Prisma Client` と `✔ Database is now in sync with your schema.`

- [ ] **Step 5: テスト用 DB にもマイグレーションを適用**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx prisma migrate deploy
```

- [ ] **Step 6: `prisma/seed.ts` を作成**

```typescript
// prisma/seed.ts
import { PrismaClient, Role } from "@prisma/client"
import bcrypt from "bcryptjs"

const prisma = new PrismaClient()

async function main() {
  const hash = (pw: string) => bcrypt.hash(pw, 10)

  await prisma.user.upsert({
    where: { email: "manager@example.com" },
    update: {},
    create: { name: "田中 部長", email: "manager@example.com", passwordHash: await hash("password"), role: Role.manager },
  })
  await prisma.user.upsert({
    where: { email: "sales@example.com" },
    update: {},
    create: { name: "鈴木 一郎", email: "sales@example.com", passwordHash: await hash("password"), role: Role.sales },
  })
  await prisma.customer.upsert({
    where: { id: BigInt(1) },
    update: {},
    create: { id: BigInt(1), name: "株式会社A", contactName: "佐藤 花子", phone: "03-1234-5678" },
  })
}

main().finally(() => prisma.$disconnect())
```

`package.json` の `prisma` セクションに追加:
```json
"prisma": {
  "seed": "ts-node prisma/seed.ts"
}
```

- [ ] **Step 7: シードを実行して確認**

```bash
npx prisma db seed
```

Expected: エラーなく完了。`npx prisma studio` で users に 2 件、customers に 1 件を目視確認。

- [ ] **Step 8: コミット**

```bash
git add prisma/ lib/
git commit -m "feat: add Prisma schema, migration, seed data, and utility functions"
```

---

## Task 3: NextAuth.js & 認証ミドルウェア

**Files:**
- Create: `lib/auth.ts`
- Create: `lib/session.ts`（Route Handler 共有ヘルパー）
- Create: `app/api/auth/[...nextauth]/route.ts`
- Create: `middleware.ts`
- Create: `app/(auth)/login/page.tsx`
- Create: `app/(app)/layout.tsx`
- Create: `app/layout.tsx`
- Create: `app/session-provider.tsx`
- Create: `types/next-auth.d.ts`
- Create: `app/(app)/page.tsx`
- Create: `__tests__/setup.ts`
- Create: `__tests__/helpers.ts`

- [ ] **Step 1: `__tests__/setup.ts` を作成**

```typescript
// __tests__/setup.ts
import { prisma } from "@/lib/prisma"

beforeEach(async () => {
  await prisma.visitRecord.deleteMany()
  await prisma.dailyReport.deleteMany()
  await prisma.customer.deleteMany()
  await prisma.user.deleteMany()
})

afterAll(async () => {
  await prisma.$disconnect()
})
```

- [ ] **Step 2: `__tests__/helpers.ts` を作成**

```typescript
// __tests__/helpers.ts
import { prisma } from "@/lib/prisma"
import { Role } from "@prisma/client"
import bcrypt from "bcryptjs"

export async function createUser(role: Role, overrides: { email?: string; name?: string } = {}) {
  return prisma.user.create({
    data: {
      name: overrides.name ?? (role === Role.manager ? "テスト部長" : "テスト営業"),
      email: overrides.email ?? `${role}-${Date.now()}@test.com`,
      passwordHash: await bcrypt.hash("password", 10),
      role,
    },
  })
}

export async function createCustomer(name = "株式会社テスト") {
  return prisma.customer.create({ data: { name } })
}

export function mockSession(userId: bigint, role: Role) {
  return { user: { id: String(userId), name: "テスト", email: "test@test.com", role } }
}
```

- [ ] **Step 3: `lib/auth.ts` を作成**

```typescript
// lib/auth.ts
import { NextAuthOptions } from "next-auth"
import CredentialsProvider from "next-auth/providers/credentials"
import bcrypt from "bcryptjs"
import { prisma } from "./prisma"

export const authOptions: NextAuthOptions = {
  providers: [
    CredentialsProvider({
      credentials: {
        email: { type: "email" },
        password: { type: "password" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials.password) return null
        const user = await prisma.user.findUnique({ where: { email: credentials.email } })
        if (!user) return null
        const valid = await bcrypt.compare(credentials.password, user.passwordHash)
        if (!valid) return null
        return { id: String(user.id), name: user.name, email: user.email, role: user.role }
      },
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.id = user.id
        token.role = (user as { role: string }).role
      }
      return token
    },
    session({ session, token }) {
      if (session.user) {
        (session.user as { id: string; role: string }).id = token.id as string
        ;(session.user as { id: string; role: string }).role = token.role as string
      }
      return session
    },
  },
  pages: { signIn: "/login" },
}
```

- [ ] **Step 4: `lib/session.ts` を作成（Route Handler 共有ヘルパー）**

```typescript
// lib/session.ts
import { getServerSession } from "next-auth"
import { authOptions } from "./auth"

export type SessionUser = { id: string; role: string; name?: string; email?: string }

export async function getSessionUser(): Promise<SessionUser | null> {
  const session = await getServerSession(authOptions)
  if (!session?.user) return null
  return session.user as SessionUser
}
```

- [ ] **Step 5: `app/api/auth/[...nextauth]/route.ts` を作成**

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth"
import { authOptions } from "@/lib/auth"

const handler = NextAuth(authOptions)
export { handler as GET, handler as POST }
```

- [ ] **Step 6: `middleware.ts` を作成**

```typescript
// middleware.ts
import { withAuth } from "next-auth/middleware"
import { NextResponse } from "next/server"

export default withAuth(
  function middleware(req) {
    const role = req.nextauth.token?.role
    const { pathname } = req.nextUrl

    if (pathname.startsWith("/users") && role !== "manager") {
      return NextResponse.redirect(new URL("/reports", req.url))
    }
    return NextResponse.next()
  },
  {
    callbacks: { authorized: ({ token }) => !!token },
  }
)

export const config = {
  matcher: ["/((?!login|api/auth|_next/static|_next/image|favicon.ico).*)"],
}
```

- [ ] **Step 7: `app/(auth)/login/page.tsx` を作成**

```tsx
// app/(auth)/login/page.tsx
"use client"
import { signIn } from "next-auth/react"
import { useRouter } from "next/navigation"
import { useState } from "react"

export default function LoginPage() {
  const router = useRouter()
  const [error, setError] = useState("")

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    const fd = new FormData(e.currentTarget)
    const result = await signIn("credentials", {
      email: fd.get("email"),
      password: fd.get("password"),
      redirect: false,
    })
    if (result?.error) {
      setError("メールアドレスまたはパスワードが正しくありません。")
    } else {
      router.push("/reports")
    }
  }

  return (
    <main style={{ maxWidth: 400, margin: "100px auto", padding: 24 }}>
      <h1>ログイン</h1>
      <form onSubmit={handleSubmit}>
        <div>
          <label>メールアドレス</label>
          <input type="email" name="email" required />
        </div>
        <div>
          <label>パスワード</label>
          <input type="password" name="password" required />
        </div>
        {error && <p style={{ color: "red" }}>{error}</p>}
        <button type="submit">ログイン</button>
      </form>
    </main>
  )
}
```

- [ ] **Step 8: `app/(app)/layout.tsx` を作成（認証済みレイアウト）**

```tsx
// app/(app)/layout.tsx
import { getServerSession } from "next-auth"
import { redirect } from "next/navigation"
import { authOptions } from "@/lib/auth"
import Link from "next/link"

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const session = await getServerSession(authOptions)
  if (!session) redirect("/login")

  const isManager = (session.user as { role: string }).role === "manager"

  return (
    <div>
      <nav>
        <Link href="/reports">日報一覧</Link>
        <Link href="/customers">顧客マスタ</Link>
        {isManager && <Link href="/users">ユーザー管理</Link>}
        <Link href="/api/auth/signout">ログアウト</Link>
      </nav>
      <main>{children}</main>
    </div>
  )
}
```

- [ ] **Step 9: `app/session-provider.tsx` と `app/layout.tsx` を作成**

```tsx
// app/session-provider.tsx
"use client"
import { SessionProvider as NextAuthSessionProvider } from "next-auth/react"
import type { Session } from "next-auth"

export default function SessionProvider({
  children,
  session,
}: {
  children: React.ReactNode
  session: Session | null
}) {
  return <NextAuthSessionProvider session={session}>{children}</NextAuthSessionProvider>
}
```

```tsx
// app/layout.tsx
import type { Metadata } from "next"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import SessionProvider from "./session-provider"

export const metadata: Metadata = { title: "営業日報システム" }

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const session = await getServerSession(authOptions)
  return (
    <html lang="ja">
      <body>
        <SessionProvider session={session}>{children}</SessionProvider>
      </body>
    </html>
  )
}
```

- [ ] **Step 10: `types/next-auth.d.ts` と `app/(app)/page.tsx` を作成**

```typescript
// types/next-auth.d.ts
import { DefaultSession } from "next-auth"

declare module "next-auth" {
  interface Session {
    user: { id: string; role: string } & DefaultSession["user"]
  }
  interface User {
    role: string
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    id: string
    role: string
  }
}
```

```typescript
// app/(app)/page.tsx
import { redirect } from "next/navigation"
export default function HomePage() {
  redirect("/reports")
}
```

- [ ] **Step 11: 開発サーバーを起動してログインを手動確認**

```bash
npm run dev
```

- `http://localhost:3000/login` で `sales@example.com` / `password` でログイン → `/reports` に遷移
- 未認証で `/reports` にアクセス → `/login` にリダイレクト
- sales で `/users` にアクセス → `/reports` にリダイレクト

- [ ] **Step 12: コミット**

```bash
git add lib/ app/ middleware.ts __tests__/ types/
git commit -m "feat: add NextAuth.js authentication, middleware, and shared session helper"
```

---

## Task 4: CI/CD & ガードレール

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.husky/pre-commit`
- Create: `.husky/pre-push`
- Create: `.lintstagedrc.json`
- Create: `Dockerfile`

- [ ] **Step 1: `.github/workflows/ci.yml` を作成**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  checks:
    name: Typecheck / Lint / Format / Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npx prettier --check .
      - run: npm run test:unit

  test-integration:
    name: Integration Tests (API)
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: daily_report_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/daily_report_test
      - run: npm test
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/daily_report_test
          NEXTAUTH_SECRET: test-secret
          NEXTAUTH_URL: http://localhost:3000
```

- [ ] **Step 2: husky を初期化**

```bash
npx husky init
```

- [ ] **Step 3: `.husky/pre-commit` を設定（フォーマット + lint-staged）**

```bash
#!/bin/sh
npx lint-staged
```

- [ ] **Step 4: `.husky/pre-push` を設定（typecheck + lint）**

```bash
#!/bin/sh
npm run typecheck && npm run lint
```

- [ ] **Step 5: `.lintstagedrc.json` を作成**

```json
{
  "*.{ts,tsx}": ["prettier --write", "eslint --fix"],
  "*.{json,md,css}": ["prettier --write"]
}
```

- [ ] **Step 6: `Dockerfile` を作成（Cloud Run 用 standalone ビルド）**

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
EXPOSE 8080
ENV PORT=8080
CMD ["node", "server.js"]
```

`.dockerignore`:
```
node_modules/
.next/
.env*.local
.env.test
```

- [ ] **Step 7: CI が通ることをローカルで確認**

```bash
npm run typecheck
npm run lint
npx prettier --check .
npm run test:unit
```

Expected: 全てエラーなく完了（test:unit は次フェーズで実装するため skip でも可）。

- [ ] **Step 8: コミット**

```bash
git add .github/ .husky/ .lintstagedrc.json Dockerfile .dockerignore
git commit -m "chore: add GitHub Actions CI, husky guardrails, and Cloud Run Dockerfile"
```

---

## Phase 1 完了チェック

- [ ] `npm run dev` → `http://localhost:3000/login` でログインできる
- [ ] `npm run typecheck` — エラーなし
- [ ] `npm run lint` — エラーなし
- [ ] `npx prettier --check .` — エラーなし
- [ ] `npx prisma studio` — users / customers テーブルにシードデータあり
- [ ] git push 時に pre-push フックが動作する

**→ 次: [Phase 2 — API Layer](2026-05-17-phase2-api.md)**
