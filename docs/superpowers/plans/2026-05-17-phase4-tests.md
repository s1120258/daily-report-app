# Phase 4: Tests & Quality

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Vitest 単体テスト・Playwright E2E テストを実装し、全テストスイートを PASS させる。

**Phases:**
- [Phase 1](2026-05-17-phase1-foundation.md) — Foundation ✅
- [Phase 2](2026-05-17-phase2-api.md) — API Layer ✅
- [Phase 3](2026-05-17-phase3-ui.md) — UI Layer ✅
- [Phase 4 (本ファイル)](2026-05-17-phase4-tests.md) — Tests & Quality

---

## Task 11: Vitest 単体テスト

**Files:**
- Create: `__tests__/unit/utils.test.ts`
- Create: `__tests__/unit/session.test.ts`

- [ ] **Step 1: 失敗するテストを書く**

```typescript
// __tests__/unit/utils.test.ts
import { describe, it, expect } from "vitest"
import { serializeId, formatDate } from "@/lib/utils"

describe("serializeId", () => {
  it("converts BigInt to string", () => {
    expect(serializeId(BigInt(123))).toBe("123")
  })

  it("handles large BigInt", () => {
    expect(serializeId(BigInt("9007199254740993"))).toBe("9007199254740993")
  })
})

describe("formatDate", () => {
  it("formats UTC date to YYYY-MM-DD", () => {
    expect(formatDate(new Date("2026-05-17T00:00:00.000Z"))).toBe("2026-05-17")
  })

  it("truncates time part", () => {
    expect(formatDate(new Date("2026-05-17T23:59:59.999Z"))).toBe("2026-05-17")
  })
})
```

```typescript
// __tests__/unit/session.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest"

vi.mock("next-auth", () => ({ getServerSession: vi.fn() }))
vi.mock("@/lib/auth", () => ({ authOptions: {} }))

import { getServerSession } from "next-auth"
import { getSessionUser } from "@/lib/session"

describe("getSessionUser", () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it("returns null when session is null", async () => {
    vi.mocked(getServerSession).mockResolvedValue(null)
    expect(await getSessionUser()).toBeNull()
  })

  it("returns null when session has no user", async () => {
    vi.mocked(getServerSession).mockResolvedValue({ expires: "" } as never)
    expect(await getSessionUser()).toBeNull()
  })

  it("returns user object from session", async () => {
    vi.mocked(getServerSession).mockResolvedValue({
      user: { id: "42", role: "sales", name: "鈴木", email: "s@test.com" },
      expires: "",
    })
    const result = await getSessionUser()
    expect(result).toEqual({ id: "42", role: "sales", name: "鈴木", email: "s@test.com" })
  })
})
```

- [ ] **Step 2: テストが FAIL することを確認**

```bash
npx vitest run __tests__/unit/
```

Expected: FAIL（`lib/utils.ts` や `lib/session.ts` のインポートエラー、または関数未実装）

- [ ] **Step 3: テストを実行して PASS することを確認**

`lib/utils.ts` と `lib/session.ts` は Phase 1 で作成済み。以下を実行:

```bash
npm run test:unit
```

Expected: PASS（全 6 テスト）

- [ ] **Step 4: コミット**

```bash
git add __tests__/unit/
git commit -m "test: add Vitest unit tests for utils and session helper"
```

---

## Task 12: Playwright E2E テスト

**Files:**
- Create: `playwright.config.ts`
- Create: `e2e/seed.ts`
- Create: `e2e/auth.spec.ts`
- Create: `e2e/daily-reports.spec.ts`
- Create: `e2e/comments.spec.ts`

- [ ] **Step 1: Playwright を設定**

```bash
npx playwright install --with-deps chromium
```

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

- [ ] **Step 2: E2E 用シードを作成**

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
      {
        name: "テスト部長",
        email: "manager@e2e.com",
        passwordHash: await bcrypt.hash("password", 10),
        role: Role.manager,
      },
      {
        name: "テスト営業",
        email: "sales@e2e.com",
        passwordHash: await bcrypt.hash("password", 10),
        role: Role.sales,
      },
    ],
  })
  await prisma.customer.create({ data: { name: "E2Eテスト株式会社" } })
  await prisma.$disconnect()
}
```

- [ ] **Step 3: `e2e/auth.spec.ts` を作成**

```typescript
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test"
import { seedE2E } from "./seed"

test.beforeAll(async () => {
  await seedE2E()
})

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

test("未認証で /reports にアクセスすると /login にリダイレクトされる", async ({ page }) => {
  await page.goto("/reports")
  await expect(page).toHaveURL(/login/)
})
```

- [ ] **Step 4: `e2e/daily-reports.spec.ts` を作成**

```typescript
// e2e/daily-reports.spec.ts
import { test, expect } from "@playwright/test"
import { seedE2E } from "./seed"

test.beforeAll(async () => {
  await seedE2E()
})

async function loginAsSales(page: Parameters<Parameters<typeof test>[1]>[0]) {
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
  await expect(page.locator("select")).toHaveCount(2)
})
```

- [ ] **Step 5: `e2e/comments.spec.ts` を作成**

```typescript
// e2e/comments.spec.ts
import { test, expect } from "@playwright/test"
import { seedE2E } from "./seed"

test.beforeAll(async () => {
  await seedE2E()
})

test("manager が Problem にコメントを保存できる", async ({ page }) => {
  // sales で日報作成
  await page.goto("/login")
  await page.fill("input[type=email]", "sales@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await page.waitForURL("/reports")
  await page.click("a[href='/reports/new']")
  await page.fill("textarea >> nth=1", "コメントテスト用の課題。")
  await page.click("button[type=submit]")
  const reportUrl = page.url()

  // manager でコメント保存
  await page.goto("/login")
  await page.fill("input[type=email]", "manager@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await page.waitForURL("/reports")
  await page.goto(reportUrl)
  await page.fill("textarea >> nth=0", "承知しました。来週対応します。")
  await page.click("button:has-text('コメント保存') >> nth=0")
  await expect(page.getByText("承知しました。来週対応します。")).toBeVisible()
})

test("sales の日報詳細画面にコメント入力欄が表示されない", async ({ page }) => {
  await page.goto("/login")
  await page.fill("input[type=email]", "sales@e2e.com")
  await page.fill("input[type=password]", "password")
  await page.click("button[type=submit]")
  await page.waitForURL("/reports")

  const links = page.locator("table tbody a")
  const count = await links.count()
  if (count > 0) {
    await links.first().click()
    await expect(page.locator("button:has-text('コメント保存')")).toHaveCount(0)
  }
})
```

- [ ] **Step 6: E2E テストを実行**

開発サーバーを起動してから実行（`webServer` 設定があるため自動起動も可）:

```bash
npx playwright test
```

Expected: 全テスト PASS

- [ ] **Step 7: `package.json` のスクリプトを確認**

Phase 1 で以下のスクリプトが設定済みであることを確認:

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

- [ ] **Step 8: コミット**

```bash
git add playwright.config.ts e2e/
git commit -m "test: add Playwright E2E tests for auth, daily reports, and comments"
```

---

## 全体完了チェック

- [ ] `npm run typecheck` — エラーなし
- [ ] `npm run lint` — エラーなし
- [ ] `npx prettier --check .` — エラーなし
- [ ] `npm run test:unit` — 全 Vitest テスト PASS
- [ ] `DATABASE_URL=... npm test` — 全 Jest 統合テスト PASS
- [ ] `npm run test:e2e` — 全 Playwright テスト PASS
- [ ] `npm run build` — ビルドエラーなし
- [ ] sales でログイン → `/users` アクセス → `/reports` にリダイレクト
- [ ] manager でログイン → 全員の日報が閲覧でき、コメントが保存できる
- [ ] 同一日に2回日報作成しようとするとエラーが表示される
- [ ] GitHub Actions で CI が PASS する（push または PR 時）
