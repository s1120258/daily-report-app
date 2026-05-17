# 営業日報システム 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 営業担当者が訪問記録・Problem/Plan を日報として記録し、マネージャーがコメントできる Next.js Web アプリを構築する。

**Architecture:** Next.js App Router で UI・API を一体管理。認証は NextAuth.js のセッションクッキー方式。権限制御は middleware（ページ）と Route Handler（API）の二重防御で実施。DB は PostgreSQL、Prisma ORM 経由でのみアクセス。

**Tech Stack:** Next.js 14（TypeScript, App Router）・PostgreSQL・Prisma・NextAuth.js・bcryptjs・Jest・Playwright

---

## ファイル構成

```
app/
  (auth)/
    login/
      page.tsx                      # SC-01 ログイン画面
  (app)/
    layout.tsx                      # 認証済みレイアウト（ナビ含む）
    reports/
      page.tsx                      # SC-02 日報一覧
      new/page.tsx                  # SC-03 日報作成
      [id]/
        page.tsx                    # SC-04 日報詳細
        edit/page.tsx               # SC-03 日報編集
    customers/
      page.tsx                      # SC-05 顧客マスタ一覧
      new/page.tsx                  # SC-06 顧客登録
      [id]/edit/page.tsx            # SC-06 顧客編集
    users/
      page.tsx                      # SC-07 ユーザー管理一覧
      new/page.tsx                  # SC-08 ユーザー登録
      [id]/edit/page.tsx            # SC-08 ユーザー編集
  api/
    auth/[...nextauth]/route.ts
    daily-reports/
      route.ts                      # GET, POST
      [id]/
        route.ts                    # GET, PUT
        problem-comment/route.ts    # PATCH
        plan-comment/route.ts       # PATCH
    customers/
      route.ts                      # GET, POST
      [id]/route.ts                 # PUT
    users/
      route.ts                      # GET, POST
      [id]/route.ts                 # PUT
  layout.tsx
  globals.css
middleware.ts
lib/
  auth.ts                           # NextAuth 設定
  prisma.ts                         # Prisma クライアント singleton
prisma/
  schema.prisma
  seed.ts
__tests__/
  setup.ts
  helpers.ts                        # セッションモック・テストデータ生成
  api/
    daily-reports.test.ts
    comments.test.ts
    customers.test.ts
    users.test.ts
e2e/
  auth.spec.ts
  daily-reports.spec.ts
  comments.spec.ts
```

---

## Task 1: プロジェクトセットアップ

**Files:**
- Create: `package.json`（Next.js プロジェクト）
- Create: `.env.local`
- Create: `.env.test`
- Create: `.gitignore`

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
npm install --save-dev @types/bcryptjs jest @types/jest ts-jest jest-environment-node @playwright/test
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

- [ ] **Step 5: `.gitignore` に `.env.local` と `.env.test` が含まれていることを確認**

```bash
grep ".env" .gitignore
```

Expected: `.env*.local` または `.env.local` がリストされている。

- [ ] **Step 6: jest.config.ts を作成**

```typescript
// jest.config.ts
import type { Config } from "jest"
import nextJest from "next/jest"

const createJestConfig = nextJest({ dir: "./" })

const config: Config = {
  testEnvironment: "node",
  setupFilesAfterFramework: ["<rootDir>/__tests__/setup.ts"],
  moduleNameMapper: { "^@/(.*)$": "<rootDir>/$1" },
  testMatch: ["**/__tests__/**/*.test.ts"],
}

export default createJestConfig(config)
```

- [ ] **Step 7: コミット**

```bash
git add -A
git commit -m "chore: initial Next.js project setup with dependencies"
```

---

## Task 2: Prisma スキーマ & マイグレーション

**Files:**
- Create: `prisma/schema.prisma`
- Create: `prisma/seed.ts`
- Create: `lib/prisma.ts`

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
  customer    Customer     @relation(fields: [customerId], references: [id])
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

- [ ] **Step 3: マイグレーションを実行**

```bash
npx prisma migrate dev --name init
```

Expected: `✔ Generated Prisma Client` と `✔ Database is now in sync with your schema.`

- [ ] **Step 4: テスト用 DB にもマイグレーションを適用**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx prisma migrate deploy
```

- [ ] **Step 5: `prisma/seed.ts` を作成**

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

- [ ] **Step 6: シードを実行して確認**

```bash
npx prisma db seed
npx prisma studio
```

Expected: `users` に2件、`customers` に1件のレコードが表示される。

- [ ] **Step 7: コミット**

```bash
git add prisma/ lib/prisma.ts jest.config.ts
git commit -m "feat: add Prisma schema, migration, and seed data"
```

---

## Task 3: NextAuth.js & 認証ミドルウェア

**Files:**
- Create: `lib/auth.ts`
- Create: `app/api/auth/[...nextauth]/route.ts`
- Create: `middleware.ts`
- Create: `app/(auth)/login/page.tsx`
- Create: `__tests__/setup.ts`
- Create: `__tests__/helpers.ts`

- [ ] **Step 1: `__tests__/setup.ts` を作成（テスト用 DB クリーンアップ）**

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

- [ ] **Step 2: `__tests__/helpers.ts` を作成（テストデータ生成・セッションモック）**

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

- [ ] **Step 4: `app/api/auth/[...nextauth]/route.ts` を作成**

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth"
import { authOptions } from "@/lib/auth"

const handler = NextAuth(authOptions)
export { handler as GET, handler as POST }
```

- [ ] **Step 5: `middleware.ts` を作成**

```typescript
// middleware.ts
import { withAuth } from "next-auth/middleware"
import { NextResponse } from "next/server"

export default withAuth(
  function middleware(req) {
    const role = req.nextauth.token?.role
    const { pathname } = req.nextUrl

    // manager 専用ページへの sales アクセスをブロック
    if ((pathname.startsWith("/users")) && role !== "manager") {
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

- [ ] **Step 6: `app/(auth)/login/page.tsx` を作成**

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

- [ ] **Step 7: `app/(app)/layout.tsx` を作成（認証済みレイアウト）**

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

- [ ] **Step 8: `app/layout.tsx`（root レイアウト）を作成**

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

`app/session-provider.tsx`:

```tsx
// app/session-provider.tsx
"use client"
import { SessionProvider as NextAuthSessionProvider } from "next-auth/react"
import type { Session } from "next-auth"

export default function SessionProvider({ children, session }: { children: React.ReactNode; session: Session | null }) {
  return <NextAuthSessionProvider session={session}>{children}</NextAuthSessionProvider>
}
```

- [ ] **Step 9: NextAuth の型拡張ファイルを作成**

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

- [ ] **Step 10: `app/(app)/page.tsx` を作成（`/reports` へリダイレクト）**

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

`http://localhost:3000/login` にアクセスし、`manager@example.com` / `password` でログインできることを確認。未認証で `/reports` にアクセスすると `/login` にリダイレクトされることを確認。

- [ ] **Step 9: コミット**

```bash
git add lib/auth.ts middleware.ts app/ __tests__/
git commit -m "feat: add NextAuth.js authentication and middleware"
```

---

## Task 4: 日報 API

**Files:**
- Create: `app/api/daily-reports/route.ts`
- Create: `app/api/daily-reports/[id]/route.ts`
- Create: `__tests__/api/daily-reports.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

```typescript
// __tests__/api/daily-reports.test.ts
import { NextRequest } from "next/server"
import { Role } from "@prisma/client"
import { prisma } from "@/lib/prisma"
import { createUser, createCustomer, mockSession } from "../helpers"
import { GET, POST } from "@/app/api/daily-reports/route"
import { GET as GET_DETAIL, PUT } from "@/app/api/daily-reports/[id]/route"
import { getServerSession } from "next-auth"

jest.mock("next-auth", () => ({ getServerSession: jest.fn() }))
const mockedSession = getServerSession as jest.Mock

describe("GET /api/daily-reports", () => {
  it("returns 401 when unauthenticated", async () => {
    mockedSession.mockResolvedValue(null)
    const req = new NextRequest("http://localhost/api/daily-reports")
    const res = await GET(req)
    expect(res.status).toBe(401)
  })

  it("sales sees only own reports", async () => {
    const salesA = await createUser(Role.sales, { email: "a@test.com" })
    const salesB = await createUser(Role.sales, { email: "b@test.com" })
    await prisma.dailyReport.create({ data: { userId: salesA.id, reportDate: new Date("2026-05-17") } })
    await prisma.dailyReport.create({ data: { userId: salesB.id, reportDate: new Date("2026-05-17") } })

    mockedSession.mockResolvedValue(mockSession(salesA.id, Role.sales))
    const req = new NextRequest("http://localhost/api/daily-reports")
    const res = await GET(req)
    const json = await res.json()

    expect(res.status).toBe(200)
    expect(json.data).toHaveLength(1)
    expect(json.data[0].user.id).toBe(String(salesA.id))
  })

  it("manager sees all reports", async () => {
    const manager = await createUser(Role.manager)
    const salesA = await createUser(Role.sales, { email: "a@test.com" })
    const salesB = await createUser(Role.sales, { email: "b@test.com" })
    await prisma.dailyReport.create({ data: { userId: salesA.id, reportDate: new Date("2026-05-17") } })
    await prisma.dailyReport.create({ data: { userId: salesB.id, reportDate: new Date("2026-05-17") } })

    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))
    const req = new NextRequest("http://localhost/api/daily-reports")
    const res = await GET(req)
    const json = await res.json()

    expect(res.status).toBe(200)
    expect(json.data).toHaveLength(2)
  })
})

describe("POST /api/daily-reports", () => {
  it("returns 403 when manager tries to create", async () => {
    const manager = await createUser(Role.manager)
    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))
    const req = new NextRequest("http://localhost/api/daily-reports", {
      method: "POST",
      body: JSON.stringify({ reportDate: "2026-05-17" }),
    })
    const res = await POST(req)
    expect(res.status).toBe(403)
  })

  it("returns 409 when duplicate report date", async () => {
    const sales = await createUser(Role.sales)
    await prisma.dailyReport.create({ data: { userId: sales.id, reportDate: new Date("2026-05-17") } })

    mockedSession.mockResolvedValue(mockSession(sales.id, Role.sales))
    const req = new NextRequest("http://localhost/api/daily-reports", {
      method: "POST",
      body: JSON.stringify({ reportDate: "2026-05-17" }),
    })
    const res = await POST(req)
    expect(res.status).toBe(409)
  })

  it("creates report with visit records", async () => {
    const sales = await createUser(Role.sales)
    const customer = await createCustomer()

    mockedSession.mockResolvedValue(mockSession(sales.id, Role.sales))
    const req = new NextRequest("http://localhost/api/daily-reports", {
      method: "POST",
      body: JSON.stringify({
        reportDate: "2026-05-17",
        problemText: "課題あり",
        planText: "明日の計画",
        visitRecords: [{ customerId: Number(customer.id), visitedAt: "2026-05-17T10:00:00+09:00", content: "初回訪問" }],
      }),
    })
    const res = await POST(req)
    const json = await res.json()

    expect(res.status).toBe(201)
    const report = await prisma.dailyReport.findUnique({
      where: { id: BigInt(json.data.id) },
      include: { visitRecords: true },
    })
    expect(report?.visitRecords).toHaveLength(1)
  })
})
```

- [ ] **Step 2: テストが失敗することを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/daily-reports.test.ts --no-coverage
```

Expected: FAIL（`Cannot find module '@/app/api/daily-reports/route'`）

- [ ] **Step 3: `app/api/daily-reports/route.ts` を実装**

```typescript
// app/api/daily-reports/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"

function getUser(session: Awaited<ReturnType<typeof getServerSession>>) {
  return session?.user as { id: string; role: string } | undefined
}

export async function GET(req: NextRequest) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })

  const { searchParams } = req.nextUrl
  const from = searchParams.get("from")
  const to = searchParams.get("to")
  const userIdParam = searchParams.get("user_id")

  const whereUserId =
    user.role === "manager" && userIdParam ? BigInt(userIdParam) : undefined

  const reports = await prisma.dailyReport.findMany({
    where: {
      userId: user.role === "sales" ? BigInt(user.id) : whereUserId,
      ...(from || to
        ? { reportDate: { gte: from ? new Date(from) : undefined, lte: to ? new Date(to) : undefined } }
        : {}),
    },
    include: {
      user: { select: { id: true, name: true } },
      _count: { select: { visitRecords: true } },
    },
    orderBy: { reportDate: "desc" },
  })

  return NextResponse.json({
    data: reports.map((r) => ({
      id: String(r.id),
      reportDate: r.reportDate.toISOString().slice(0, 10),
      user: { id: String(r.user.id), name: r.user.name },
      visitCount: r._count.visitRecords,
      hasProblemComment: !!r.problemCommentText,
      hasPlanComment: !!r.planCommentText,
    })),
  })
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "sales") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const body = await req.json()
  const { reportDate, problemText, planText, visitRecords = [] } = body

  if (!reportDate) return NextResponse.json({ error: "reportDate is required" }, { status: 400 })

  const existing = await prisma.dailyReport.findUnique({
    where: { userId_reportDate: { userId: BigInt(user.id), reportDate: new Date(reportDate) } },
  })
  if (existing) return NextResponse.json({ error: "この日付の日報は既に作成されています。" }, { status: 409 })

  // Validate customer IDs
  if (visitRecords.length > 0) {
    const customerIds = visitRecords.map((v: { customerId: number }) => BigInt(v.customerId))
    const customers = await prisma.customer.findMany({ where: { id: { in: customerIds } } })
    if (customers.length !== customerIds.length) {
      return NextResponse.json({ error: "Invalid customerId" }, { status: 400 })
    }
  }

  const report = await prisma.dailyReport.create({
    data: {
      userId: BigInt(user.id),
      reportDate: new Date(reportDate),
      problemText,
      planText,
      visitRecords: {
        create: visitRecords.map((v: { customerId: number; visitedAt: string; content: string }) => ({
          customerId: BigInt(v.customerId),
          visitedAt: new Date(v.visitedAt),
          content: v.content,
        })),
      },
    },
  })

  return NextResponse.json({ data: { id: String(report.id) } }, { status: 201 })
}
```

- [ ] **Step 4: `app/api/daily-reports/[id]/route.ts` を実装**

```typescript
// app/api/daily-reports/[id]/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"

type Params = { params: { id: string } }

function getUser(session: Awaited<ReturnType<typeof getServerSession>>) {
  return session?.user as { id: string; role: string } | undefined
}

export async function GET(_req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })

  const report = await prisma.dailyReport.findUnique({
    where: { id: BigInt(params.id) },
    include: {
      user: { select: { id: true, name: true } },
      problemCommenter: { select: { id: true, name: true } },
      planCommenter: { select: { id: true, name: true } },
      visitRecords: { include: { customer: { select: { id: true, name: true } } }, orderBy: { visitedAt: "asc" } },
    },
  })

  if (!report) return NextResponse.json({ error: "Not found" }, { status: 404 })
  if (user.role === "sales" && String(report.userId) !== user.id) {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 })
  }

  return NextResponse.json({
    data: {
      id: String(report.id),
      reportDate: report.reportDate.toISOString().slice(0, 10),
      user: { id: String(report.user.id), name: report.user.name },
      problemText: report.problemText,
      planText: report.planText,
      problemComment: report.problemCommentText
        ? { text: report.problemCommentText, commentedBy: { id: String(report.problemCommenter!.id), name: report.problemCommenter!.name }, commentedAt: report.problemCommentedAt }
        : null,
      planComment: report.planCommentText
        ? { text: report.planCommentText, commentedBy: { id: String(report.planCommenter!.id), name: report.planCommenter!.name }, commentedAt: report.planCommentedAt }
        : null,
      visitRecords: report.visitRecords.map((v) => ({
        id: String(v.id),
        customer: { id: String(v.customer.id), name: v.customer.name },
        visitedAt: v.visitedAt,
        content: v.content,
      })),
      createdAt: report.createdAt,
      updatedAt: report.updatedAt,
    },
  })
}

export async function PUT(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "sales") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const report = await prisma.dailyReport.findUnique({ where: { id: BigInt(params.id) } })
  if (!report) return NextResponse.json({ error: "Not found" }, { status: 404 })
  if (String(report.userId) !== user.id) return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const body = await req.json()
  const { problemText, planText, visitRecords = [] } = body

  // visitRecords 洗い替え
  await prisma.$transaction([
    prisma.visitRecord.deleteMany({ where: { dailyReportId: BigInt(params.id) } }),
    prisma.dailyReport.update({
      where: { id: BigInt(params.id) },
      data: {
        problemText,
        planText,
        visitRecords: {
          create: visitRecords.map((v: { customerId: number; visitedAt: string; content: string }) => ({
            customerId: BigInt(v.customerId),
            visitedAt: new Date(v.visitedAt),
            content: v.content,
          })),
        },
      },
    }),
  ])

  return NextResponse.json({ data: { id: params.id } })
}
```

- [ ] **Step 5: テストを実行してパスすることを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/daily-reports.test.ts --no-coverage
```

Expected: PASS（全テスト緑）

- [ ] **Step 6: コミット**

```bash
git add app/api/daily-reports/ __tests__/api/daily-reports.test.ts __tests__/
git commit -m "feat: add daily reports API with tests"
```

---

## Task 5: コメント API

**Files:**
- Create: `app/api/daily-reports/[id]/problem-comment/route.ts`
- Create: `app/api/daily-reports/[id]/plan-comment/route.ts`
- Create: `__tests__/api/comments.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

```typescript
// __tests__/api/comments.test.ts
import { NextRequest } from "next/server"
import { Role } from "@prisma/client"
import { prisma } from "@/lib/prisma"
import { createUser, mockSession } from "../helpers"
import { PATCH as PATCH_PROBLEM } from "@/app/api/daily-reports/[id]/problem-comment/route"
import { PATCH as PATCH_PLAN } from "@/app/api/daily-reports/[id]/plan-comment/route"
import { getServerSession } from "next-auth"

jest.mock("next-auth", () => ({ getServerSession: jest.fn() }))
const mockedSession = getServerSession as jest.Mock

async function createReport(userId: bigint) {
  return prisma.dailyReport.create({ data: { userId, reportDate: new Date("2026-05-17") } })
}

describe("PATCH /api/daily-reports/[id]/problem-comment", () => {
  it("returns 403 when sales tries to comment", async () => {
    const sales = await createUser(Role.sales)
    const report = await createReport(sales.id)
    mockedSession.mockResolvedValue(mockSession(sales.id, Role.sales))

    const req = new NextRequest(`http://localhost/api/daily-reports/${report.id}/problem-comment`, {
      method: "PATCH",
      body: JSON.stringify({ text: "コメント" }),
    })
    const res = await PATCH_PROBLEM(req, { params: { id: String(report.id) } })
    expect(res.status).toBe(403)
  })

  it("manager can add problem comment", async () => {
    const manager = await createUser(Role.manager)
    const sales = await createUser(Role.sales, { email: "s@test.com" })
    const report = await createReport(sales.id)
    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))

    const req = new NextRequest(`http://localhost/api/daily-reports/${report.id}/problem-comment`, {
      method: "PATCH",
      body: JSON.stringify({ text: "来週一緒に訪問しましょう。" }),
    })
    const res = await PATCH_PROBLEM(req, { params: { id: String(report.id) } })
    const json = await res.json()

    expect(res.status).toBe(200)
    expect(json.data.text).toBe("来週一緒に訪問しましょう。")
    expect(json.data.commentedBy.id).toBe(String(manager.id))

    const updated = await prisma.dailyReport.findUnique({ where: { id: report.id } })
    expect(updated?.problemCommentText).toBe("来週一緒に訪問しましょう。")
  })
})
```

- [ ] **Step 2: テストが失敗することを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/comments.test.ts --no-coverage
```

Expected: FAIL

- [ ] **Step 3: `app/api/daily-reports/[id]/problem-comment/route.ts` を実装**

```typescript
// app/api/daily-reports/[id]/problem-comment/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"

type Params = { params: { id: string } }

export async function PATCH(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions)
  const user = session?.user as { id: string; role: string } | undefined
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const report = await prisma.dailyReport.findUnique({ where: { id: BigInt(params.id) } })
  if (!report) return NextResponse.json({ error: "Not found" }, { status: 404 })

  const { text } = await req.json()
  const updated = await prisma.dailyReport.update({
    where: { id: BigInt(params.id) },
    data: { problemCommentText: text, problemCommentBy: BigInt(user.id), problemCommentedAt: new Date() },
    include: { problemCommenter: { select: { id: true, name: true } } },
  })

  return NextResponse.json({
    data: {
      text: updated.problemCommentText,
      commentedBy: { id: String(updated.problemCommenter!.id), name: updated.problemCommenter!.name },
      commentedAt: updated.problemCommentedAt,
    },
  })
}
```

- [ ] **Step 4: `app/api/daily-reports/[id]/plan-comment/route.ts` を実装（同じ構造で `plan` カラムを使用）**

```typescript
// app/api/daily-reports/[id]/plan-comment/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"

type Params = { params: { id: string } }

export async function PATCH(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions)
  const user = session?.user as { id: string; role: string } | undefined
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const report = await prisma.dailyReport.findUnique({ where: { id: BigInt(params.id) } })
  if (!report) return NextResponse.json({ error: "Not found" }, { status: 404 })

  const { text } = await req.json()
  const updated = await prisma.dailyReport.update({
    where: { id: BigInt(params.id) },
    data: { planCommentText: text, planCommentBy: BigInt(user.id), planCommentedAt: new Date() },
    include: { planCommenter: { select: { id: true, name: true } } },
  })

  return NextResponse.json({
    data: {
      text: updated.planCommentText,
      commentedBy: { id: String(updated.planCommenter!.id), name: updated.planCommenter!.name },
      commentedAt: updated.planCommentedAt,
    },
  })
}
```

- [ ] **Step 5: テストを実行してパスすることを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/comments.test.ts --no-coverage
```

Expected: PASS

- [ ] **Step 6: コミット**

```bash
git add app/api/daily-reports/[id]/problem-comment/ app/api/daily-reports/[id]/plan-comment/ __tests__/api/comments.test.ts
git commit -m "feat: add problem-comment and plan-comment APIs with tests"
```

---

## Task 6: 顧客 API

**Files:**
- Create: `app/api/customers/route.ts`
- Create: `app/api/customers/[id]/route.ts`
- Create: `__tests__/api/customers.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

```typescript
// __tests__/api/customers.test.ts
import { NextRequest } from "next/server"
import { Role } from "@prisma/client"
import { createUser, createCustomer, mockSession } from "../helpers"
import { GET, POST } from "@/app/api/customers/route"
import { PUT } from "@/app/api/customers/[id]/route"
import { getServerSession } from "next-auth"

jest.mock("next-auth", () => ({ getServerSession: jest.fn() }))
const mockedSession = getServerSession as jest.Mock

describe("GET /api/customers", () => {
  it("returns customers for any authenticated user", async () => {
    const sales = await createUser(Role.sales)
    await createCustomer("株式会社A")
    await createCustomer("株式会社B")
    mockedSession.mockResolvedValue(mockSession(sales.id, Role.sales))

    const res = await GET(new NextRequest("http://localhost/api/customers"))
    const json = await res.json()
    expect(res.status).toBe(200)
    expect(json.data).toHaveLength(2)
  })
})

describe("POST /api/customers", () => {
  it("returns 403 when sales tries to create", async () => {
    const sales = await createUser(Role.sales)
    mockedSession.mockResolvedValue(mockSession(sales.id, Role.sales))
    const res = await POST(new NextRequest("http://localhost/api/customers", {
      method: "POST",
      body: JSON.stringify({ name: "株式会社C" }),
    }))
    expect(res.status).toBe(403)
  })

  it("manager can create customer", async () => {
    const manager = await createUser(Role.manager)
    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))
    const res = await POST(new NextRequest("http://localhost/api/customers", {
      method: "POST",
      body: JSON.stringify({ name: "株式会社D", contactName: "山田 太郎" }),
    }))
    expect(res.status).toBe(201)
  })
})
```

- [ ] **Step 2: テストが失敗することを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/customers.test.ts --no-coverage
```

Expected: FAIL

- [ ] **Step 3: `app/api/customers/route.ts` を実装**

```typescript
// app/api/customers/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"

function getUser(session: Awaited<ReturnType<typeof getServerSession>>) {
  return session?.user as { id: string; role: string } | undefined
}

export async function GET(_req: NextRequest) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })

  const customers = await prisma.customer.findMany({ orderBy: { name: "asc" } })
  return NextResponse.json({
    data: customers.map((c) => ({
      id: String(c.id),
      name: c.name,
      contactName: c.contactName,
      phone: c.phone,
      email: c.email,
    })),
  })
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const { name, contactName, phone, email } = await req.json()
  if (!name) return NextResponse.json({ error: "name is required" }, { status: 400 })

  const customer = await prisma.customer.create({ data: { name, contactName, phone, email } })
  return NextResponse.json({ data: { id: String(customer.id) } }, { status: 201 })
}
```

- [ ] **Step 4: `app/api/customers/[id]/route.ts` を実装**

```typescript
// app/api/customers/[id]/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"

type Params = { params: { id: string } }

export async function PUT(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions)
  const user = session?.user as { id: string; role: string } | undefined
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const existing = await prisma.customer.findUnique({ where: { id: BigInt(params.id) } })
  if (!existing) return NextResponse.json({ error: "Not found" }, { status: 404 })

  const { name, contactName, phone, email } = await req.json()
  if (!name) return NextResponse.json({ error: "name is required" }, { status: 400 })

  await prisma.customer.update({ where: { id: BigInt(params.id) }, data: { name, contactName, phone, email } })
  return NextResponse.json({ data: { id: params.id } })
}
```

- [ ] **Step 5: テストを実行してパスすることを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/customers.test.ts --no-coverage
```

Expected: PASS

- [ ] **Step 6: コミット**

```bash
git add app/api/customers/ __tests__/api/customers.test.ts
git commit -m "feat: add customers API with tests"
```

---

## Task 7: ユーザー API

**Files:**
- Create: `app/api/users/route.ts`
- Create: `app/api/users/[id]/route.ts`
- Create: `__tests__/api/users.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

```typescript
// __tests__/api/users.test.ts
import { NextRequest } from "next/server"
import { Role } from "@prisma/client"
import { createUser, mockSession } from "../helpers"
import { GET, POST } from "@/app/api/users/route"
import { PUT } from "@/app/api/users/[id]/route"
import { getServerSession } from "next-auth"

jest.mock("next-auth", () => ({ getServerSession: jest.fn() }))
const mockedSession = getServerSession as jest.Mock

describe("GET /api/users", () => {
  it("returns 403 for sales", async () => {
    const sales = await createUser(Role.sales)
    mockedSession.mockResolvedValue(mockSession(sales.id, Role.sales))
    const res = await GET(new NextRequest("http://localhost/api/users"))
    expect(res.status).toBe(403)
  })

  it("returns all users for manager", async () => {
    const manager = await createUser(Role.manager)
    await createUser(Role.sales, { email: "s@test.com" })
    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))
    const res = await GET(new NextRequest("http://localhost/api/users"))
    const json = await res.json()
    expect(res.status).toBe(200)
    expect(json.data).toHaveLength(2)
  })
})

describe("POST /api/users", () => {
  it("returns 409 on duplicate email", async () => {
    const manager = await createUser(Role.manager, { email: "manager@test.com" })
    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))
    const res = await POST(new NextRequest("http://localhost/api/users", {
      method: "POST",
      body: JSON.stringify({ name: "重複", email: "manager@test.com", password: "pw", role: "sales" }),
    }))
    expect(res.status).toBe(409)
  })

  it("manager can create user", async () => {
    const manager = await createUser(Role.manager)
    mockedSession.mockResolvedValue(mockSession(manager.id, Role.manager))
    const res = await POST(new NextRequest("http://localhost/api/users", {
      method: "POST",
      body: JSON.stringify({ name: "新規営業", email: "new@test.com", password: "password", role: "sales" }),
    }))
    expect(res.status).toBe(201)
  })
})
```

- [ ] **Step 2: テストが失敗することを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/users.test.ts --no-coverage
```

Expected: FAIL

- [ ] **Step 3: `app/api/users/route.ts` を実装**

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import bcrypt from "bcryptjs"
import { Role } from "@prisma/client"

function getUser(session: Awaited<ReturnType<typeof getServerSession>>) {
  return session?.user as { id: string; role: string } | undefined
}

export async function GET(_req: NextRequest) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const users = await prisma.user.findMany({ orderBy: { name: "asc" } })
  return NextResponse.json({
    data: users.map((u) => ({ id: String(u.id), name: u.name, email: u.email, role: u.role })),
  })
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  const user = getUser(session)
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const { name, email, password, role } = await req.json()
  if (!name || !email || !password || !role) {
    return NextResponse.json({ error: "name, email, password, role are required" }, { status: 400 })
  }

  const existing = await prisma.user.findUnique({ where: { email } })
  if (existing) return NextResponse.json({ error: "このメールアドレスは既に使用されています。" }, { status: 409 })

  const created = await prisma.user.create({
    data: { name, email, passwordHash: await bcrypt.hash(password, 10), role: role as Role },
  })
  return NextResponse.json({ data: { id: String(created.id) } }, { status: 201 })
}
```

- [ ] **Step 4: `app/api/users/[id]/route.ts` を実装**

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from "next/server"
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import bcrypt from "bcryptjs"
import { Role } from "@prisma/client"

type Params = { params: { id: string } }

export async function PUT(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions)
  const user = session?.user as { id: string; role: string } | undefined
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  if (user.role !== "manager") return NextResponse.json({ error: "Forbidden" }, { status: 403 })

  const existing = await prisma.user.findUnique({ where: { id: BigInt(params.id) } })
  if (!existing) return NextResponse.json({ error: "Not found" }, { status: 404 })

  const { name, email, password, role } = await req.json()
  if (!name || !email || !role) {
    return NextResponse.json({ error: "name, email, role are required" }, { status: 400 })
  }

  await prisma.user.update({
    where: { id: BigInt(params.id) },
    data: {
      name,
      email,
      role: role as Role,
      ...(password ? { passwordHash: await bcrypt.hash(password, 10) } : {}),
    },
  })
  return NextResponse.json({ data: { id: params.id } })
}
```

- [ ] **Step 5: テストを実行してパスすることを確認**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/users.test.ts --no-coverage
```

Expected: PASS

- [ ] **Step 6: 全 API テストをまとめて実行**

```bash
DATABASE_URL="postgresql://postgres:password@localhost:5432/daily_report_test" \
  npx jest __tests__/api/ --no-coverage
```

Expected: 全テスト PASS

- [ ] **Step 7: コミット**

```bash
git add app/api/users/ __tests__/api/users.test.ts
git commit -m "feat: add users API with tests"
```

---

## Task 8: 日報画面（SC-02 / SC-03 / SC-04）

**Files:**
- Create: `app/(app)/reports/page.tsx`
- Create: `app/(app)/reports/new/page.tsx`
- Create: `app/(app)/reports/[id]/page.tsx`
- Create: `app/(app)/reports/[id]/edit/page.tsx`

- [ ] **Step 1: `app/(app)/reports/page.tsx`（日報一覧 SC-02）を作成**

```tsx
// app/(app)/reports/page.tsx
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import Link from "next/link"

export default async function ReportsPage() {
  const session = await getServerSession(authOptions)
  const user = session!.user as { id: string; role: string }
  const isManager = user.role === "manager"

  const reports = await prisma.dailyReport.findMany({
    where: isManager ? {} : { userId: BigInt(user.id) },
    include: {
      user: { select: { name: true } },
      _count: { select: { visitRecords: true } },
    },
    orderBy: { reportDate: "desc" },
  })

  return (
    <div>
      <h1>日報一覧</h1>
      {!isManager && <Link href="/reports/new">新規作成</Link>}
      <table>
        <thead>
          <tr>
            <th>日付</th>
            {isManager && <th>担当者</th>}
            <th>訪問件数</th>
            <th>Problem</th>
            <th>Plan</th>
          </tr>
        </thead>
        <tbody>
          {reports.map((r) => (
            <tr key={String(r.id)}>
              <td>
                <Link href={`/reports/${r.id}`}>
                  {r.reportDate.toISOString().slice(0, 10)}
                </Link>
              </td>
              {isManager && <td>{r.user.name}</td>}
              <td>{r._count.visitRecords}</td>
              <td>{r.problemCommentText ? "✓" : "-"}</td>
              <td>{r.planCommentText ? "✓" : "-"}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

- [ ] **Step 2: `app/(app)/reports/new/page.tsx`（日報作成 SC-03）を作成**

```tsx
// app/(app)/reports/new/page.tsx
import { getServerSession } from "next-auth"
import { redirect } from "next/navigation"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import ReportForm from "./form"

export default async function NewReportPage() {
  const session = await getServerSession(authOptions)
  const user = session!.user as { id: string; role: string }
  if (user.role !== "sales") redirect("/reports")

  const customers = await prisma.customer.findMany({ orderBy: { name: "asc" } })
  return <ReportForm customers={customers.map((c) => ({ id: String(c.id), name: c.name }))} />
}
```

`app/(app)/reports/new/form.tsx`（Client Component）を作成:

```tsx
// app/(app)/reports/new/form.tsx
"use client"
import { useRouter } from "next/navigation"
import { useState } from "react"

type Customer = { id: string; name: string }
type VisitRecord = { customerId: string; visitedAt: string; content: string }

export default function ReportForm({ customers }: { customers: Customer[] }) {
  const router = useRouter()
  const [visits, setVisits] = useState<VisitRecord[]>([{ customerId: "", visitedAt: "", content: "" }])
  const [problem, setProblem] = useState("")
  const [plan, setPlan] = useState("")
  const [error, setError] = useState("")

  const today = new Date().toISOString().slice(0, 10)

  function addVisit() {
    setVisits((v) => [...v, { customerId: "", visitedAt: "", content: "" }])
  }

  function removeVisit(i: number) {
    setVisits((v) => v.filter((_, idx) => idx !== i))
  }

  function updateVisit(i: number, field: keyof VisitRecord, value: string) {
    setVisits((v) => v.map((row, idx) => (idx === i ? { ...row, [field]: value } : row)))
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    const res = await fetch("/api/daily-reports", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        reportDate: today,
        problemText: problem,
        planText: plan,
        visitRecords: visits
          .filter((v) => v.customerId && v.content)
          .map((v) => ({ customerId: Number(v.customerId), visitedAt: v.visitedAt, content: v.content })),
      }),
    })
    if (res.status === 409) { setError("本日の日報は既に作成されています。"); return }
    if (!res.ok) { setError("保存に失敗しました。"); return }
    const json = await res.json()
    router.push(`/reports/${json.data.id}`)
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>日報作成（{today}）</h1>
      {error && <p style={{ color: "red" }}>{error}</p>}

      <h2>訪問記録</h2>
      {visits.map((v, i) => (
        <div key={i} style={{ border: "1px solid #ccc", padding: 8, marginBottom: 8 }}>
          <select value={v.customerId} onChange={(e) => updateVisit(i, "customerId", e.target.value)} required>
            <option value="">顧客を選択</option>
            {customers.map((c) => <option key={c.id} value={c.id}>{c.name}</option>)}
          </select>
          <input type="datetime-local" value={v.visitedAt} onChange={(e) => updateVisit(i, "visitedAt", e.target.value)} required />
          <textarea value={v.content} onChange={(e) => updateVisit(i, "content", e.target.value)} placeholder="訪問内容" required />
          <button type="button" onClick={() => removeVisit(i)}>削除</button>
        </div>
      ))}
      <button type="button" onClick={addVisit}>行追加</button>

      <h2>Problem（課題・相談）</h2>
      <textarea value={problem} onChange={(e) => setProblem(e.target.value)} rows={4} style={{ width: "100%" }} />

      <h2>Plan（明日やること）</h2>
      <textarea value={plan} onChange={(e) => setPlan(e.target.value)} rows={4} style={{ width: "100%" }} />

      <div>
        <button type="submit">保存</button>
        <button type="button" onClick={() => router.push("/reports")}>キャンセル</button>
      </div>
    </form>
  )
}
```

- [ ] **Step 3: `app/(app)/reports/[id]/page.tsx`（日報詳細 SC-04）を作成**

```tsx
// app/(app)/reports/[id]/page.tsx
import { getServerSession } from "next-auth"
import { notFound, redirect } from "next/navigation"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import Link from "next/link"
import CommentSection from "./comment-section"

export default async function ReportDetailPage({ params }: { params: { id: string } }) {
  const session = await getServerSession(authOptions)
  const user = session!.user as { id: string; role: string }

  const report = await prisma.dailyReport.findUnique({
    where: { id: BigInt(params.id) },
    include: {
      user: { select: { id: true, name: true } },
      problemCommenter: { select: { name: true } },
      planCommenter: { select: { name: true } },
      visitRecords: { include: { customer: { select: { name: true } } }, orderBy: { visitedAt: "asc" } },
    },
  })

  if (!report) notFound()
  if (user.role === "sales" && String(report.userId) !== user.id) redirect("/reports")

  const isManager = user.role === "manager"
  const isOwner = String(report.userId) === user.id

  return (
    <div>
      <h1>日報詳細</h1>
      <p>{report.reportDate.toISOString().slice(0, 10)} — {report.user.name}</p>

      {isOwner && <Link href={`/reports/${params.id}/edit`}>編集</Link>}
      <Link href="/reports">一覧へ戻る</Link>

      <h2>訪問記録</h2>
      <table>
        <thead><tr><th>顧客</th><th>訪問日時</th><th>訪問内容</th></tr></thead>
        <tbody>
          {report.visitRecords.map((v) => (
            <tr key={String(v.id)}>
              <td>{v.customer.name}</td>
              <td>{v.visitedAt.toLocaleString("ja-JP")}</td>
              <td>{v.content}</td>
            </tr>
          ))}
        </tbody>
      </table>

      <h2>Problem（課題・相談）</h2>
      <p>{report.problemText || "（未記入）"}</p>
      <CommentSection
        reportId={params.id}
        type="problem"
        commentText={report.problemCommentText}
        commenterName={report.problemCommenter?.name}
        commentedAt={report.problemCommentedAt?.toISOString()}
        isManager={isManager}
      />

      <h2>Plan（明日やること）</h2>
      <p>{report.planText || "（未記入）"}</p>
      <CommentSection
        reportId={params.id}
        type="plan"
        commentText={report.planCommentText}
        commenterName={report.planCommenter?.name}
        commentedAt={report.planCommentedAt?.toISOString()}
        isManager={isManager}
      />
    </div>
  )
}
```

`app/(app)/reports/[id]/comment-section.tsx`（Client Component）を作成:

```tsx
// app/(app)/reports/[id]/comment-section.tsx
"use client"
import { useState } from "react"
import { useRouter } from "next/navigation"

type Props = {
  reportId: string
  type: "problem" | "plan"
  commentText: string | null | undefined
  commenterName: string | null | undefined
  commentedAt: string | null | undefined
  isManager: boolean
}

export default function CommentSection({ reportId, type, commentText, commenterName, commentedAt, isManager }: Props) {
  const router = useRouter()
  const [text, setText] = useState(commentText ?? "")
  const [saving, setSaving] = useState(false)

  async function handleSave() {
    setSaving(true)
    await fetch(`/api/daily-reports/${reportId}/${type}-comment`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text }),
    })
    setSaving(false)
    router.refresh()
  }

  return (
    <div>
      {commentText && (
        <blockquote>
          <p>{commentText}</p>
          <footer>{commenterName} — {commentedAt ? new Date(commentedAt).toLocaleString("ja-JP") : ""}</footer>
        </blockquote>
      )}
      {isManager && (
        <div>
          <textarea value={text} onChange={(e) => setText(e.target.value)} rows={3} style={{ width: "100%" }} />
          <button onClick={handleSave} disabled={saving}>
            {saving ? "保存中..." : "コメント保存"}
          </button>
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 4: `app/(app)/reports/[id]/edit/page.tsx`（日報編集）を作成**

```tsx
// app/(app)/reports/[id]/edit/page.tsx
import { getServerSession } from "next-auth"
import { notFound, redirect } from "next/navigation"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import EditForm from "./form"

export default async function EditReportPage({ params }: { params: { id: string } }) {
  const session = await getServerSession(authOptions)
  const user = session!.user as { id: string; role: string }
  if (user.role !== "sales") redirect("/reports")

  const report = await prisma.dailyReport.findUnique({
    where: { id: BigInt(params.id) },
    include: { visitRecords: true },
  })
  if (!report) notFound()
  if (String(report.userId) !== user.id) redirect("/reports")

  const customers = await prisma.customer.findMany({ orderBy: { name: "asc" } })

  return (
    <EditForm
      reportId={params.id}
      initialProblem={report.problemText ?? ""}
      initialPlan={report.planText ?? ""}
      initialVisits={report.visitRecords.map((v) => ({
        customerId: String(v.customerId),
        visitedAt: v.visitedAt.toISOString().slice(0, 16),
        content: v.content,
      }))}
      customers={customers.map((c) => ({ id: String(c.id), name: c.name }))}
    />
  )
}
```

`app/(app)/reports/[id]/edit/form.tsx`（Client Component）を作成:

```tsx
// app/(app)/reports/[id]/edit/form.tsx
"use client"
import { useRouter } from "next/navigation"
import { useState } from "react"

type Customer = { id: string; name: string }
type VisitRecord = { customerId: string; visitedAt: string; content: string }

type Props = {
  reportId: string
  initialProblem: string
  initialPlan: string
  initialVisits: VisitRecord[]
  customers: Customer[]
}

export default function EditForm({ reportId, initialProblem, initialPlan, initialVisits, customers }: Props) {
  const router = useRouter()
  const [visits, setVisits] = useState<VisitRecord[]>(initialVisits.length > 0 ? initialVisits : [{ customerId: "", visitedAt: "", content: "" }])
  const [problem, setProblem] = useState(initialProblem)
  const [plan, setPlan] = useState(initialPlan)

  function addVisit() { setVisits((v) => [...v, { customerId: "", visitedAt: "", content: "" }]) }
  function removeVisit(i: number) { setVisits((v) => v.filter((_, idx) => idx !== i)) }
  function updateVisit(i: number, field: keyof VisitRecord, value: string) {
    setVisits((v) => v.map((row, idx) => (idx === i ? { ...row, [field]: value } : row)))
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    await fetch(`/api/daily-reports/${reportId}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        problemText: problem,
        planText: plan,
        visitRecords: visits
          .filter((v) => v.customerId && v.content)
          .map((v) => ({ customerId: Number(v.customerId), visitedAt: v.visitedAt, content: v.content })),
      }),
    })
    router.push(`/reports/${reportId}`)
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>日報編集</h1>
      <h2>訪問記録</h2>
      {visits.map((v, i) => (
        <div key={i} style={{ border: "1px solid #ccc", padding: 8, marginBottom: 8 }}>
          <select value={v.customerId} onChange={(e) => updateVisit(i, "customerId", e.target.value)} required>
            <option value="">顧客を選択</option>
            {customers.map((c) => <option key={c.id} value={c.id}>{c.name}</option>)}
          </select>
          <input type="datetime-local" value={v.visitedAt} onChange={(e) => updateVisit(i, "visitedAt", e.target.value)} required />
          <textarea value={v.content} onChange={(e) => updateVisit(i, "content", e.target.value)} required />
          <button type="button" onClick={() => removeVisit(i)}>削除</button>
        </div>
      ))}
      <button type="button" onClick={addVisit}>行追加</button>
      <h2>Problem</h2>
      <textarea value={problem} onChange={(e) => setProblem(e.target.value)} rows={4} style={{ width: "100%" }} />
      <h2>Plan</h2>
      <textarea value={plan} onChange={(e) => setPlan(e.target.value)} rows={4} style={{ width: "100%" }} />
      <div>
        <button type="submit">保存</button>
        <button type="button" onClick={() => router.push(`/reports/${reportId}`)}>キャンセル</button>
      </div>
    </form>
  )
}
```

- [ ] **Step 5: 開発サーバーで動作確認**

```bash
npm run dev
```

- `http://localhost:3000/reports` を開き、日報一覧が表示されることを確認
- 「新規作成」から日報を作成し、詳細画面に遷移することを確認
- Manager でログインし、コメントが保存できることを確認

- [ ] **Step 6: コミット**

```bash
git add app/\(app\)/reports/
git commit -m "feat: add daily report screens (list, create, detail, edit)"
```

---

## Task 9: マスタデータ画面（SC-05〜SC-08）

**Files:**
- Create: `app/(app)/customers/page.tsx`
- Create: `app/(app)/customers/new/page.tsx`
- Create: `app/(app)/customers/[id]/edit/page.tsx`
- Create: `app/(app)/users/page.tsx`
- Create: `app/(app)/users/new/page.tsx`
- Create: `app/(app)/users/[id]/edit/page.tsx`

- [ ] **Step 1: 顧客マスタ一覧 `app/(app)/customers/page.tsx`（SC-05）を作成**

```tsx
// app/(app)/customers/page.tsx
import { getServerSession } from "next-auth"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import Link from "next/link"

export default async function CustomersPage() {
  const session = await getServerSession(authOptions)
  const user = session!.user as { role: string }
  const isManager = user.role === "manager"

  const customers = await prisma.customer.findMany({ orderBy: { name: "asc" } })

  return (
    <div>
      <h1>顧客マスタ</h1>
      {isManager && <Link href="/customers/new">新規登録</Link>}
      <table>
        <thead>
          <tr><th>顧客名</th><th>担当者名</th><th>電話番号</th><th>メール</th>{isManager && <th>操作</th>}</tr>
        </thead>
        <tbody>
          {customers.map((c) => (
            <tr key={String(c.id)}>
              <td>{c.name}</td><td>{c.contactName}</td><td>{c.phone}</td><td>{c.email}</td>
              {isManager && <td><Link href={`/customers/${c.id}/edit`}>編集</Link></td>}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

- [ ] **Step 2: 顧客登録フォーム `app/(app)/customers/new/page.tsx`（SC-06）を作成**

```tsx
// app/(app)/customers/new/page.tsx
"use client"
import { useRouter } from "next/navigation"
import { useState } from "react"

export default function NewCustomerPage() {
  const router = useRouter()
  const [form, setForm] = useState({ name: "", contactName: "", phone: "", email: "" })

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    await fetch("/api/customers", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    })
    router.push("/customers")
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>顧客登録</h1>
      <input placeholder="顧客名（必須）" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} required />
      <input placeholder="担当者名" value={form.contactName} onChange={(e) => setForm({ ...form, contactName: e.target.value })} />
      <input placeholder="電話番号" value={form.phone} onChange={(e) => setForm({ ...form, phone: e.target.value })} />
      <input placeholder="メール" value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} />
      <button type="submit">保存</button>
      <button type="button" onClick={() => router.push("/customers")}>キャンセル</button>
    </form>
  )
}
```

- [ ] **Step 3: 顧客編集 `app/(app)/customers/[id]/edit/page.tsx` を作成**

```tsx
// app/(app)/customers/[id]/edit/page.tsx
import { notFound } from "next/navigation"
import { prisma } from "@/lib/prisma"
import CustomerEditForm from "./form"

export default async function EditCustomerPage({ params }: { params: { id: string } }) {
  const customer = await prisma.customer.findUnique({ where: { id: BigInt(params.id) } })
  if (!customer) notFound()
  return (
    <CustomerEditForm
      id={params.id}
      initial={{ name: customer.name, contactName: customer.contactName ?? "", phone: customer.phone ?? "", email: customer.email ?? "" }}
    />
  )
}
```

`app/(app)/customers/[id]/edit/form.tsx`:

```tsx
// app/(app)/customers/[id]/edit/form.tsx
"use client"
import { useRouter } from "next/navigation"
import { useState } from "react"

type FormData = { name: string; contactName: string; phone: string; email: string }

export default function CustomerEditForm({ id, initial }: { id: string; initial: FormData }) {
  const router = useRouter()
  const [form, setForm] = useState(initial)

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    await fetch(`/api/customers/${id}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    })
    router.push("/customers")
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>顧客編集</h1>
      <input placeholder="顧客名（必須）" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} required />
      <input placeholder="担当者名" value={form.contactName} onChange={(e) => setForm({ ...form, contactName: e.target.value })} />
      <input placeholder="電話番号" value={form.phone} onChange={(e) => setForm({ ...form, phone: e.target.value })} />
      <input placeholder="メール" value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} />
      <button type="submit">保存</button>
      <button type="button" onClick={() => router.push("/customers")}>キャンセル</button>
    </form>
  )
}
```

- [ ] **Step 4: ユーザー管理一覧 `app/(app)/users/page.tsx`（SC-07）を作成**

```tsx
// app/(app)/users/page.tsx
import { getServerSession } from "next-auth"
import { redirect } from "next/navigation"
import { authOptions } from "@/lib/auth"
import { prisma } from "@/lib/prisma"
import Link from "next/link"

export default async function UsersPage() {
  const session = await getServerSession(authOptions)
  const user = session!.user as { role: string }
  if (user.role !== "manager") redirect("/reports")

  const users = await prisma.user.findMany({ orderBy: { name: "asc" } })

  return (
    <div>
      <h1>ユーザー管理</h1>
      <Link href="/users/new">新規登録</Link>
      <table>
        <thead><tr><th>氏名</th><th>メール</th><th>ロール</th><th>操作</th></tr></thead>
        <tbody>
          {users.map((u) => (
            <tr key={String(u.id)}>
              <td>{u.name}</td><td>{u.email}</td>
              <td>{u.role === "manager" ? "マネージャー" : "営業"}</td>
              <td><Link href={`/users/${u.id}/edit`}>編集</Link></td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

- [ ] **Step 5: ユーザー登録 `app/(app)/users/new/page.tsx`（SC-08）を作成**

```tsx
// app/(app)/users/new/page.tsx
"use client"
import { useRouter } from "next/navigation"
import { useState } from "react"

export default function NewUserPage() {
  const router = useRouter()
  const [form, setForm] = useState({ name: "", email: "", password: "", role: "sales" })
  const [error, setError] = useState("")

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    const res = await fetch("/api/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    })
    if (res.status === 409) { setError("このメールアドレスは既に使用されています。"); return }
    router.push("/users")
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>ユーザー登録</h1>
      {error && <p style={{ color: "red" }}>{error}</p>}
      <input placeholder="氏名（必須）" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} required />
      <input type="email" placeholder="メールアドレス（必須）" value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} required />
      <input type="password" placeholder="パスワード（必須）" value={form.password} onChange={(e) => setForm({ ...form, password: e.target.value })} required />
      <select value={form.role} onChange={(e) => setForm({ ...form, role: e.target.value })}>
        <option value="sales">営業</option>
        <option value="manager">マネージャー</option>
      </select>
      <button type="submit">保存</button>
      <button type="button" onClick={() => router.push("/users")}>キャンセル</button>
    </form>
  )
}
```

- [ ] **Step 6: ユーザー編集 `app/(app)/users/[id]/edit/page.tsx` を作成**

```tsx
// app/(app)/users/[id]/edit/page.tsx
import { notFound } from "next/navigation"
import { prisma } from "@/lib/prisma"
import UserEditForm from "./form"

export default async function EditUserPage({ params }: { params: { id: string } }) {
  const user = await prisma.user.findUnique({ where: { id: BigInt(params.id) } })
  if (!user) notFound()
  return (
    <UserEditForm
      id={params.id}
      initial={{ name: user.name, email: user.email, role: user.role }}
    />
  )
}
```

`app/(app)/users/[id]/edit/form.tsx`:

```tsx
// app/(app)/users/[id]/edit/form.tsx
"use client"
import { useRouter } from "next/navigation"
import { useState } from "react"

export default function UserEditForm({ id, initial }: { id: string; initial: { name: string; email: string; role: string } }) {
  const router = useRouter()
  const [form, setForm] = useState({ ...initial, password: "" })

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    await fetch(`/api/users/${id}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    })
    router.push("/users")
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>ユーザー編集</h1>
      <input placeholder="氏名（必須）" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} required />
      <input type="email" placeholder="メールアドレス（必須）" value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} required />
      <input type="password" placeholder="パスワード（空欄で変更なし）" value={form.password} onChange={(e) => setForm({ ...form, password: e.target.value })} />
      <select value={form.role} onChange={(e) => setForm({ ...form, role: e.target.value })}>
        <option value="sales">営業</option>
        <option value="manager">マネージャー</option>
      </select>
      <button type="submit">保存</button>
      <button type="button" onClick={() => router.push("/users")}>キャンセル</button>
    </form>
  )
}
```

- [ ] **Step 7: 開発サーバーで全画面を動作確認**

```bash
npm run dev
```

- manager でログイン → 顧客マスタ一覧・登録・編集を確認
- manager でログイン → ユーザー管理一覧・登録・編集を確認
- sales でログイン → `/users` にアクセスすると `/reports` にリダイレクトされることを確認

- [ ] **Step 8: コミット**

```bash
git add app/\(app\)/customers/ app/\(app\)/users/
git commit -m "feat: add master data screens (customers and users)"
```

---

## Task 10: E2E テスト（Playwright）

**Files:**
- Create: `playwright.config.ts`
- Create: `e2e/auth.spec.ts`
- Create: `e2e/daily-reports.spec.ts`
- Create: `e2e/comments.spec.ts`

- [ ] **Step 1: Playwright を設定**

```bash
npx playwright install --with-deps chromium
```

`playwright.config.ts`:

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test"

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: false,
  retries: 0,
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
  },
  projects: [{ name: "chromium", use: { ...devices["Desktop Chrome"] } }],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: true,
  },
})
```

- [ ] **Step 2: E2E 用シードスクリプトを追加**

`e2e/seed.ts`（E2E テスト実行前に呼ぶ）:

```typescript
// e2e/seed.ts
import { PrismaClient, Role } from "@prisma/client"
import bcrypt from "bcryptjs"

const prisma = new PrismaClient()

export async function seedE2E() {
  await prisma.visitRecord.deleteMany()
  await prisma.dailyReport.deleteMany()
  await prisma.customer.deleteMany()
  await prisma.user.deleteMany()

  await prisma.user.createMany({
    data: [
      { name: "テスト部長", email: "manager@e2e.com", passwordHash: await bcrypt.hash("password", 10), role: Role.manager },
      { name: "テスト営業", email: "sales@e2e.com", passwordHash: await bcrypt.hash("password", 10), role: Role.sales },
    ],
  })
  await prisma.customer.create({ data: { name: "E2Eテスト株式会社" } })
}
```

- [ ] **Step 3: 認証 E2E テストを書く**

```typescript
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test"

test("正常ログイン後に日報一覧へ遷移する", async ({ page }) => {
  await page.goto("/login")
  await page.fill("input[type=email]", "sales@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await expect(page).toHaveURL("/reports")
})

test("誤ったパスワードでエラーメッセージが表示される", async ({ page }) => {
  await page.goto("/login")
  await page.fill("input[type=email]", "sales@e2e.com")
  await page.fill("input[type=password]", "wrongpassword")
  await page.click("button[type=submit]")
  await expect(page.getByText("メールアドレスまたはパスワードが正しくありません")).toBeVisible()
})

test("未認証で / にアクセスすると /login にリダイレクトされる", async ({ page }) => {
  await page.goto("/reports")
  await expect(page).toHaveURL(/login/)
})
```

- [ ] **Step 4: 日報作成 E2E テストを書く**

```typescript
// e2e/daily-reports.spec.ts
import { test, expect } from "@playwright/test"

async function loginAsSales(page: Parameters<typeof test>[1]) {
  await page.goto("/login")
  await page.fill("input[type=email]", "sales@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await page.waitForURL("/reports")
}

test("日報を新規作成して詳細画面に遷移する", async ({ page }) => {
  await loginAsSales(page)
  await page.click("a[href='/reports/new']")
  await page.selectOption("select", { label: "E2Eテスト株式会社" })
  await page.fill("input[type=datetime-local]", "2026-05-17T10:00")
  await page.fill("textarea >> nth=0", "初回訪問で担当者と面談。")
  await page.fill("textarea >> nth=1", "予算確保が課題。")
  await page.fill("textarea >> nth=2", "来週フォローアップ。")
  await page.click("button[type=submit]")
  await expect(page).toHaveURL(/\/reports\/\d+$/)
  await expect(page.getByText("初回訪問で担当者と面談。")).toBeVisible()
})

test("行追加ボタンで訪問記録を複数行追加できる", async ({ page }) => {
  await loginAsSales(page)
  await page.click("a[href='/reports/new']")
  await page.click("button:has-text('行追加')")
  const rows = page.locator("select")
  await expect(rows).toHaveCount(2)
})
```

- [ ] **Step 5: コメント E2E テストを書く**

```typescript
// e2e/comments.spec.ts
import { test, expect } from "@playwright/test"

async function loginAsManager(page: Parameters<typeof test>[1]) {
  await page.goto("/login")
  await page.fill("input[type=email]", "manager@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await page.waitForURL("/reports")
}

test("manager が Problem にコメントを保存できる", async ({ page }) => {
  // 事前に sales でログインして日報作成
  await page.goto("/login")
  await page.fill("input[type=email]", "sales@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await page.waitForURL("/reports")
  await page.click("a[href='/reports/new']")
  await page.fill("textarea >> nth=1", "コメントテスト用の課題。")
  await page.click("button[type=submit]")
  const reportUrl = page.url()

  // manager でコメント
  await loginAsManager(page)
  await page.goto(reportUrl)
  await page.fill("textarea >> nth=0", "承知しました。来週対応します。")
  await page.click("button:has-text('コメント保存') >> nth=0")
  await expect(page.getByText("承知しました。来週対応します。")).toBeVisible()
})
```

- [ ] **Step 6: E2E テストを実行**

```bash
npx playwright test
```

Expected: 全テスト PASS（サーバーが起動している必要がある）

- [ ] **Step 7: `package.json` にテストスクリプトを追加**

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "test": "DATABASE_URL='postgresql://postgres:password@localhost:5432/daily_report_test' jest --no-coverage",
  "test:e2e": "playwright test"
}
```

- [ ] **Step 8: コミット**

```bash
git add playwright.config.ts e2e/ package.json
git commit -m "feat: add Playwright E2E tests for auth, daily reports, and comments"
```

---

## 完了チェック

実装完了後に以下をすべて確認する。

- [ ] `npm test` — 全統合テスト PASS
- [ ] `npm run test:e2e` — 全 E2E テスト PASS
- [ ] `npm run build` — ビルドエラーなし
- [ ] sales でログイン → `/users` アクセス → `/reports` にリダイレクト
- [ ] manager でログイン → 全員の日報が閲覧でき、コメントが保存できる
- [ ] 同一日に2回日報作成しようとするとエラーが表示される
