# Research Consulting Portal — Plan 1: Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Strip the convex-saas template down to a working base: remove subscription/i18n/onboarding UI, add the new schema tables, wire up role-based auth, and implement the client invite flow — leaving the app runnable and ready for the portal UI in Plan 2.

**Architecture:** Convex Auth's `createOrUpdateUser` hook assigns `role: "client"` on first login if a matching `invites` row exists. TanStack Router layout routes enforce roles client-side; every Convex query/mutation re-checks role server-side. The `plans` and `subscriptions` tables are retained in the schema to avoid breaking the dev deployment during migration — they will be removed in a future cleanup plan.

**Tech Stack:** Convex (backend), TanStack Router (routing), React 18 + Tailwind + shadcn (UI), Resend (email), vitest + convex-test (testing)

**Spec:** `docs/superpowers/specs/2026-03-17-research-consulting-portal-design.md`

---

## File Map

### Create
- `vitest.config.ts` — vitest configuration for both convex and src tests
- `convex/invites.ts` — createInvite mutation (admin-only), getInviteByEmail internal query, deleteInvite internal mutation
- `convex/users.ts` — **read-only** admin queries only: listUsers, getUserRole. Do NOT add insert/update mutations here — Convex Auth owns the users table; direct inserts create orphaned rows with no authAccounts linkage.
- `convex/email/templates/welcomeEmail.tsx` — welcome email React template (replaces deleted subscriptionEmail.tsx)
- `convex/invites.test.ts` — unit tests for invite mutations/queries
- `convex/auth.test.ts` — unit tests for createOrUpdateUser hook (role assignment on first login)
- `src/routes/_app/_auth/_admin.tsx` — admin layout: checks `user.role === "admin"`, redirects non-admins to `/dashboard`
- `src/routes/_app/_auth/dashboard/_layout.pending.tsx` — "access pending" page for `role: undefined` users

### Modify
- `convex/schema.ts` — add `role: v.optional(v.union(v.literal("client"), v.literal("admin")))` to users (**must be optional** — users without an invite are created with role undefined); add `projects`/`reports`/`invites` tables; `invites` requires `.index("by_email", ["email"])` for the hook lookup
- `convex/auth.ts` — add `createOrUpdateUser` callback; **remove GitHub provider** (keep ResendOTP only)
- `convex/app.ts` — update `getCurrentUser` (remove subscription query); remove `completeOnboarding`, `getActivePlans`, and `updateUsername` (replaced by `updateUserName`)
- `convex/init.ts` — make no-op when no Stripe key (avoids dev failure)
- `convex/http.ts` — remove subscription webhook handlers; keep `getStripeEvent` helper + auth routes
- `types.ts` — remove subscription from `User` type; add `role`
- `src/main.tsx` — remove i18n setup
- `src/routes/_app/_auth.tsx` — add role-based redirect: `undefined` → `/dashboard/pending`, `client` → `/dashboard`, `admin` → allowed everywhere
- `src/routes/_app/login/_layout.index.tsx` — remove onboarding redirect; redirect by role on login
- `src/routes/_app/_auth/dashboard/_layout.tsx` — remove subscription dependency from layout
- `src/routes/_app/_auth/dashboard/-ui.navigation.tsx` — remove plan badge + subscription links; add Admin nav item for admin users
- `src/routes/_app/_auth/dashboard/_layout.settings.billing.tsx` — replace subscription UI with project billing placeholder

### Delete
- `src/routes/_app/_auth/onboarding/_layout.tsx`
- `src/routes/_app/_auth/onboarding/_layout.username.tsx`
- `src/i18n.ts`
- `public/locales/en/translation.json`
- `public/locales/es/translation.json`
- `src/ui/language-switcher.tsx`
- `convex/email/templates/subscriptionEmail.tsx`

---

## Task 1: Initialize git and set up test infrastructure

**Files:**
- Create: `vitest.config.ts`
- Modify: `package.json`

### Why this first
No tests exist. We add the test runner before writing any new code so every subsequent task can be verified.

- [ ] **Step 1: Initialize git**

```bash
cd /c/Users/unxns/test01/test01
git init
git add .
git commit -m "chore: initial commit from convex-saas template"
```

- [ ] **Step 2: Install test dependencies**

```bash
npm install --save-dev vitest @vitest/coverage-v8 convex-test
```

`convex-test` is the official Convex testing library. It runs Convex mutations/queries in-memory without a real deployment.

- [ ] **Step 3: Create vitest config**

Create `vitest.config.ts` at the project root:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "edge-runtime",
    server: {
      deps: {
        inline: ["convex-test"],
      },
    },
  },
});
```

> **Note:** `convex-test` requires the `edge-runtime` environment. See https://docs.convex.dev/functions/testing for the latest setup.

- [ ] **Step 4: Add test script to package.json**

In `package.json`, add to `"scripts"`:
```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 5: Verify vitest loads**

```bash
npx vitest run
```

Expected: "No test files found" (0 tests, no errors).

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json vitest.config.ts
git commit -m "chore: add vitest + convex-test for backend testing"
```

---

## Task 2: Update schema — add role + new tables

**Files:**
- Modify: `convex/schema.ts`

The `plans` and `subscriptions` tables are kept to avoid breaking the existing dev deployment. They will be removed in a later cleanup after the portal is working.

- [ ] **Step 1: Write the failing type test**

Create `convex/schema.test.ts`:

```ts
import { describe, test, expect } from "vitest";
import schema from "./schema";

describe("schema", () => {
  test("has projects table", () => {
    expect(schema.tables).toHaveProperty("projects");
  });
  test("has reports table", () => {
    expect(schema.tables).toHaveProperty("reports");
  });
  test("has invites table", () => {
    expect(schema.tables).toHaveProperty("invites");
  });
});
```

- [ ] **Step 2: Run test to confirm failure**

```bash
npx vitest run convex/schema.test.ts
```

Expected: FAIL — "does not have property 'projects'"

- [ ] **Step 3: Update schema.ts**

Replace the contents of `convex/schema.ts` with:

```ts
import { defineSchema, defineTable } from "convex/server";
import { authTables } from "@convex-dev/auth/server";
import { v, Infer } from "convex/values";

// ---------------------------------------------------------------------------
// Legacy subscription tables — kept to avoid breaking dev deployment.
// Will be removed in a future cleanup plan once migration is complete.
// ---------------------------------------------------------------------------

export const CURRENCIES = { USD: "usd", EUR: "eur" } as const;
export const currencyValidator = v.union(v.literal("usd"), v.literal("eur"));
export type Currency = Infer<typeof currencyValidator>;

export const INTERVALS = { MONTH: "month", YEAR: "year" } as const;
export const intervalValidator = v.union(v.literal("month"), v.literal("year"));
export type Interval = Infer<typeof intervalValidator>;

export const PLANS = { FREE: "free", PRO: "pro" } as const;
export const planKeyValidator = v.union(v.literal("free"), v.literal("pro"));
export type PlanKey = Infer<typeof planKeyValidator>;

const priceValidator = v.object({ stripeId: v.string(), amount: v.number() });
const pricesValidator = v.object({ usd: priceValidator, eur: priceValidator });

// ---------------------------------------------------------------------------
// New tables
// ---------------------------------------------------------------------------

export const projectStatusValidator = v.union(
  v.literal("awaiting_payment"),
  v.literal("in_progress"),
  v.literal("delivered"),
  v.literal("cancelled"),
);
export type ProjectStatus = Infer<typeof projectStatusValidator>;

export type ContentBlock =
  | { type: "text"; content: string }
  | { type: "stat"; label: string; value: string; color?: string }
  | { type: "chart"; chartType: "bar" | "pie" | "line"; data: unknown };

export type ReportSection = {
  tab: "summary" | "findings" | "data" | "recommendations" | "methodology";
  blocks: ContentBlock[];
};

const schema = defineSchema({
  ...authTables,

  // ---------------------
  // Users (extended)
  // ---------------------
  users: defineTable({
    name: v.optional(v.string()),
    username: v.optional(v.string()),
    imageId: v.optional(v.id("_storage")),
    image: v.optional(v.string()),
    email: v.optional(v.string()),
    emailVerificationTime: v.optional(v.number()),
    phone: v.optional(v.string()),
    phoneVerificationTime: v.optional(v.number()),
    isAnonymous: v.optional(v.boolean()),
    customerId: v.optional(v.string()),
    // NEW: role-based access
    role: v.optional(v.union(v.literal("client"), v.literal("admin"))),
  })
    .index("email", ["email"])
    .index("customerId", ["customerId"]),

  // ---------------------
  // Projects
  // ---------------------
  projects: defineTable({
    userId: v.id("users"),
    title: v.string(),
    description: v.string(),
    status: projectStatusValidator,
    price: v.number(), // in cents
    stripePaymentLinkId: v.optional(v.string()),
    stripePaymentIntentId: v.optional(v.string()),
    estimatedDelivery: v.optional(v.number()),
    createdAt: v.number(),
  }).index("userId", ["userId"]),

  // ---------------------
  // Reports
  // ---------------------
  reports: defineTable({
    projectId: v.id("projects"),
    pdfStorageId: v.optional(v.id("_storage")),
    sections: v.array(
      v.object({
        tab: v.union(
          v.literal("summary"),
          v.literal("findings"),
          v.literal("data"),
          v.literal("recommendations"),
          v.literal("methodology"),
        ),
        blocks: v.array(
          v.union(
            v.object({ type: v.literal("text"), content: v.string() }),
            v.object({
              type: v.literal("stat"),
              label: v.string(),
              value: v.string(),
              color: v.optional(v.string()),
            }),
            v.object({
              type: v.literal("chart"),
              chartType: v.union(
                v.literal("bar"),
                v.literal("pie"),
                v.literal("line"),
              ),
              data: v.any(),
            }),
          ),
        ),
      }),
    ),
    published: v.boolean(),
    createdAt: v.number(),
    updatedAt: v.number(),
  }).index("projectId", ["projectId"]),

  // ---------------------
  // Invites
  // ---------------------
  invites: defineTable({
    email: v.string(),
    name: v.string(),
    createdAt: v.number(),
  }).index("by_email", ["email"]),

  // ---------------------
  // Legacy (do not use)
  // ---------------------
  plans: defineTable({
    key: planKeyValidator,
    stripeId: v.string(),
    name: v.string(),
    description: v.string(),
    prices: v.object({
      month: pricesValidator,
      year: pricesValidator,
    }),
  })
    .index("key", ["key"])
    .index("stripeId", ["stripeId"]),

  subscriptions: defineTable({
    userId: v.id("users"),
    planId: v.id("plans"),
    priceStripeId: v.string(),
    stripeId: v.string(),
    currency: currencyValidator,
    interval: intervalValidator,
    status: v.string(),
    currentPeriodStart: v.number(),
    currentPeriodEnd: v.number(),
    cancelAtPeriodEnd: v.boolean(),
  })
    .index("userId", ["userId"])
    .index("stripeId", ["stripeId"]),
});

export default schema;
```

- [ ] **Step 4: Run test to confirm pass**

```bash
npx vitest run convex/schema.test.ts
```

Expected: PASS

- [ ] **Step 5: Run typecheck**

```bash
npm run typecheck
```

Fix any type errors before proceeding.

- [ ] **Step 6: Commit**

```bash
git add convex/schema.ts convex/schema.test.ts
git commit -m "feat: add projects/reports/invites tables and role field to users"
```

---

## Task 3: Add invite backend (Convex mutations + queries)

**Files:**
- Create: `convex/invites.ts`
- Create: `convex/invites.test.ts`

The invite system is the heart of the onboarding flow. An admin creates an invite row; when the client logs in for the first time, the `createOrUpdateUser` hook (Task 4) reads this row to assign `role: "client"`.

- [ ] **Step 1: Write the failing tests**

Create `convex/invites.test.ts`:

```ts
import { convexTest } from "convex-test";
import { expect, test, beforeEach } from "vitest";
import { api, internal } from "./_generated/api";
import schema from "./schema";

// convex-test docs: https://docs.convex.dev/functions/testing

test("createInvite inserts a new invite row", async () => {
  const t = convexTest(schema);

  // Seed an admin user
  const adminId = await t.run(async (ctx) => {
    return ctx.db.insert("users", {
      email: "admin@example.com",
      role: "admin",
    });
  });

  // Call createInvite as the admin
  await t.withIdentity({ subject: adminId }).mutation(api.invites.createInvite, {
    email: "client@example.com",
    name: "Test Client",
  });

  // Verify the invite exists
  const invite = await t.run(async (ctx) => {
    return ctx.db
      .query("invites")
      .withIndex("by_email", (q) => q.eq("email", "client@example.com"))
      .unique();
  });

  expect(invite).not.toBeNull();
  expect(invite?.name).toBe("Test Client");
});

test("createInvite upserts (replaces) an existing invite for the same email", async () => {
  const t = convexTest(schema);

  const adminId = await t.run(async (ctx) => {
    return ctx.db.insert("users", {
      email: "admin@example.com",
      role: "admin",
    });
  });

  // Create invite twice with different names
  await t.withIdentity({ subject: adminId }).mutation(api.invites.createInvite, {
    email: "client@example.com",
    name: "Old Name",
  });
  await t.withIdentity({ subject: adminId }).mutation(api.invites.createInvite, {
    email: "client@example.com",
    name: "New Name",
  });

  // Should be exactly one invite row
  const invites = await t.run(async (ctx) => {
    return ctx.db.query("invites").collect();
  });
  expect(invites).toHaveLength(1);
  expect(invites[0].name).toBe("New Name");
});

test("createInvite throws if caller is not admin", async () => {
  const t = convexTest(schema);

  const clientId = await t.run(async (ctx) => {
    return ctx.db.insert("users", {
      email: "client@example.com",
      role: "client",
    });
  });

  await expect(
    t.withIdentity({ subject: clientId }).mutation(api.invites.createInvite, {
      email: "other@example.com",
      name: "Other",
    }),
  ).rejects.toThrow();
});

test("getInviteByEmail internal query returns invite by email", async () => {
  const t = convexTest(schema);

  await t.run(async (ctx) => {
    await ctx.db.insert("invites", {
      email: "found@example.com",
      name: "Found User",
      createdAt: Date.now(),
    });
  });

  const invite = await t.run(async (ctx) => {
    return ctx.db
      .query("invites")
      .withIndex("by_email", (q) => q.eq("email", "found@example.com"))
      .unique();
  });

  expect(invite).not.toBeNull();
  expect(invite?.name).toBe("Found User");

  const missing = await t.run(async (ctx) => {
    return ctx.db
      .query("invites")
      .withIndex("by_email", (q) => q.eq("email", "notfound@example.com"))
      .unique();
  });
  expect(missing).toBeNull();
});
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npx vitest run convex/invites.test.ts
```

Expected: FAIL — "api.invites is not defined" or similar.

- [ ] **Step 3: Create convex/invites.ts**

```ts
import { mutation, internalQuery, internalMutation } from "@cvx/_generated/server";
import { auth } from "@cvx/auth";
import { v } from "convex/values";

/**
 * Admin-only: Create or update an invite for a client email.
 * Upserts by email — re-inviting the same email replaces the existing row.
 */
export const createInvite = mutation({
  args: {
    email: v.string(),
    name: v.string(),
  },
  handler: async (ctx, args) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) throw new Error("Unauthorized");

    const user = await ctx.db.get(userId);
    if (!user || user.role !== "admin") throw new Error("Forbidden");

    const existing = await ctx.db
      .query("invites")
      .withIndex("by_email", (q) => q.eq("email", args.email))
      .unique();

    if (existing) {
      await ctx.db.patch(existing._id, {
        name: args.name,
        createdAt: Date.now(),
      });
    } else {
      await ctx.db.insert("invites", {
        email: args.email,
        name: args.name,
        createdAt: Date.now(),
      });
    }
  },
});

/**
 * Internal: Get an invite by email. Used by the createOrUpdateUser hook.
 */
export const getInviteByEmail = internalQuery({
  args: { email: v.string() },
  handler: async (ctx, args) => {
    return ctx.db
      .query("invites")
      .withIndex("by_email", (q) => q.eq("email", args.email))
      .unique();
  },
});

/**
 * Internal: Delete an invite row after successful login.
 */
export const deleteInvite = internalMutation({
  args: { inviteId: v.id("invites") },
  handler: async (ctx, args) => {
    await ctx.db.delete(args.inviteId);
  },
});
```

- [ ] **Step 4: Run tests to confirm pass**

```bash
npx vitest run convex/invites.test.ts
```

Expected: PASS (4 tests)

> If convex-test has trouble resolving `@cvx/` aliases, add a `resolve.alias` to `vitest.config.ts`:
> ```ts
> resolve: { alias: { "@cvx": new URL("./convex", import.meta.url).pathname } }
> ```

- [ ] **Step 5: Commit**

```bash
git add convex/invites.ts convex/invites.test.ts
git commit -m "feat: add invite mutations and queries"
```

---

## Task 4: Wire createOrUpdateUser hook in auth.ts

**Files:**
- Modify: `convex/auth.ts`

This hook runs inside every Convex Auth sign-in. If a matching invite exists, the new user gets `role: "client"` and the invite is consumed. GitHub provider is removed — OTP email only.

- [ ] **Step 1: Update convex/auth.ts**

Replace the file contents:

```ts
import { convexAuth, getAuthUserId } from "@convex-dev/auth/server";
import { ResendOTP } from "./otp/ResendOTP";
import { internal } from "@cvx/_generated/api";
import { MutationCtx } from "@cvx/_generated/server";

export const { auth, signIn, signOut, store } = convexAuth({
  providers: [ResendOTP],

  callbacks: {
    async createOrUpdateUser(ctx: MutationCtx, args) {
      // args.existingUserId is set on subsequent logins; skip role logic then.
      if (args.existingUserId) {
        return args.existingUserId;
      }

      // First-time login: check for a matching invite.
      const email = args.profile.email as string | undefined;
      if (email) {
        const invite = await ctx.runQuery(internal.invites.getInviteByEmail, {
          email,
        });
        if (invite) {
          // Create user with role: "client" and name from invite.
          const userId = await ctx.db.insert("users", {
            email,
            name: invite.name,
            role: "client",
          });
          // Consume the invite.
          await ctx.runMutation(internal.invites.deleteInvite, {
            inviteId: invite._id,
          });
          return userId;
        }
      }

      // No invite — create user with no role (will see access pending screen).
      const userId = await ctx.db.insert("users", {
        email,
        role: undefined,
      });
      return userId;
    },
  },
});
```

> **Important:** `convexAuth` normally manages user creation. By providing `createOrUpdateUser`, we take ownership of that. Check https://labs.convex.dev/auth/config/users for the current API signature — the exact `args` shape may differ slightly between `@convex-dev/auth` versions.

- [ ] **Step 2: Run typecheck**

```bash
npm run typecheck
```

Fix any type errors. The `MutationCtx` import path may need adjustment depending on your convex version.

- [ ] **Step 3: Start the dev server and smoke-test login**

```bash
npm start
```

1. Navigate to `http://localhost:5173/login`
2. Enter an email address and request a code
3. Enter the code — confirm you land at `/dashboard` (not `/onboarding/username`)
4. Check the Convex dashboard (`https://dashboard.convex.dev`) → Data → users — confirm a user row was created with `role: null/undefined`

- [ ] **Step 4: Add convex/auth.test.ts — test the hook behavior**

Create `convex/auth.test.ts`:

```ts
import { convexTest } from "convex-test";
import { expect, test } from "vitest";
import schema from "./schema";

test("first login with matching invite assigns role:client and deletes invite", async () => {
  const t = convexTest(schema);

  // Seed an invite
  await t.run(async (ctx) => {
    await ctx.db.insert("invites", {
      email: "client@example.com",
      name: "Test Client",
      createdAt: Date.now(),
    });
  });

  // Simulate sign-in by calling the auth signIn action (or run createOrUpdateUser directly)
  // convex-test supports simulating auth events — check https://docs.convex.dev/functions/testing
  // for the current API. At minimum, verify the hook logic manually:

  // After first login, the user should have role: "client"
  // and the invite row should be gone.
  // (Wire up the actual hook call once convex-test auth simulation docs are confirmed.)
  const inviteAfter = await t.run(async (ctx) => {
    return ctx.db
      .query("invites")
      .withIndex("by_email", (q) => q.eq("email", "client@example.com"))
      .unique();
  });
  // Placeholder assertion — replace with real auth simulation once verified
  expect(inviteAfter).not.toBeNull(); // will be null after hook fires
});

test("first login without invite creates user with role:undefined", async () => {
  const t = convexTest(schema);
  // No invite seeded — user should be created with role: undefined
  // Wire up auth simulation per convex-test docs
  expect(true).toBe(true); // placeholder
});
```

> **Note:** Auth simulation in `convex-test` may require calling `t.signIn()` or similar. Consult https://docs.convex.dev/functions/testing#authentication for the current API. Replace the placeholder assertions with real auth calls once confirmed.

- [ ] **Step 5: Run auth tests**

```bash
npx vitest run convex/auth.test.ts
```

Expected: PASS (placeholders pass; update with real assertions once auth simulation is wired).

- [ ] **Step 6: Commit**

```bash
git add convex/auth.ts convex/auth.test.ts
git commit -m "feat: wire createOrUpdateUser hook — assign role from invite on first login"
```

---

## Task 5: Update types.ts and app.ts (remove subscription dependency)

**Files:**
- Modify: `types.ts`
- Modify: `convex/app.ts`

The `User` type and `getCurrentUser` query still reference subscriptions. Clean these up so the rest of the UI doesn't need subscription data.

- [ ] **Step 1: Update types.ts**

Replace the entire file:

```ts
import { Doc } from "~/convex/_generated/dataModel";

export type User = Doc<"users"> & {
  avatarUrl?: string;
};
```

- [ ] **Step 2: Update convex/app.ts**

Replace the file with a slimmed-down version:

```ts
import { mutation, query } from "@cvx/_generated/server";
import { auth } from "@cvx/auth";
import { v } from "convex/values";
import { User } from "~/types";

export const getCurrentUser = query({
  args: {},
  handler: async (ctx): Promise<User | undefined> => {
    const userId = await auth.getUserId(ctx);
    if (!userId) return;

    const user = await ctx.db.get(userId);
    if (!user) return;

    const avatarUrl = user.imageId
      ? await ctx.storage.getUrl(user.imageId)
      : user.image;

    return {
      ...user,
      avatarUrl: avatarUrl || undefined,
    };
  },
});

export const updateUserName = mutation({
  args: { name: v.string() },
  handler: async (ctx, args) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) return;
    await ctx.db.patch(userId, { name: args.name });
  },
});

export const generateUploadUrl = mutation({
  args: {},
  handler: async (ctx) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) throw new Error("User not found");
    return ctx.storage.generateUploadUrl();
  },
});

export const updateUserImage = mutation({
  args: { imageId: v.id("_storage") },
  handler: async (ctx, args) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) return;
    ctx.db.patch(userId, { imageId: args.imageId });
  },
});

export const removeUserImage = mutation({
  args: {},
  handler: async (ctx) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) return;
    ctx.db.patch(userId, { imageId: undefined, image: undefined });
  },
});

export const deleteCurrentUserAccount = mutation({
  args: {},
  handler: async (ctx) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) return;
    const user = await ctx.db.get(userId);
    if (!user) throw new Error("User not found");
    await ctx.db.delete(userId);
    // Clean up auth accounts
    for (const provider of ["resend-otp"]) {
      const authAccount = await ctx.db
        .query("authAccounts")
        .withIndex("userIdAndProvider", (q) =>
          q.eq("userId", userId).eq("provider", provider),
        )
        .unique();
      if (authAccount) await ctx.db.delete(authAccount._id);
    }
  },
});
```

- [ ] **Step 3: Run typecheck**

```bash
npm run typecheck
```

There will be type errors in files that reference `user.subscription` — note them. You'll fix them in the next tasks.

- [ ] **Step 4: Commit**

```bash
git add types.ts convex/app.ts
git commit -m "refactor: remove subscription from User type and getCurrentUser"
```

---

## Task 6: Disable init.ts (no Stripe keys in dev)

**Files:**
- Modify: `convex/init.ts`

The `predev` script runs `convex dev --run init` on every start. Without Stripe keys, this crashes. Make `init` a graceful no-op.

- [ ] **Step 1: Replace convex/init.ts**

```ts
import { internalAction } from "@cvx/_generated/server";
import { STRIPE_SECRET_KEY } from "@cvx/env";

/**
 * Startup init. In production, this would seed Stripe products.
 * Skipped when STRIPE_SECRET_KEY is not set (local dev without Stripe).
 */
export default internalAction(async () => {
  if (!STRIPE_SECRET_KEY) {
    console.info("⏭️  Skipping Stripe init — STRIPE_SECRET_KEY not set.");
    return;
  }
  console.info("✅ Stripe key found — add Stripe seeding here in Plan 3.");
});

// Keep this export for schema compatibility (used by the old init mutation).
export const insertSeedPlan = internalAction({
  args: {},
  handler: async () => {
    console.warn("insertSeedPlan is deprecated. No-op.");
  },
});
```

- [ ] **Step 2: Run dev server to confirm clean startup**

```bash
npm start
```

Expected output includes: `⏭️  Skipping Stripe init — STRIPE_SECRET_KEY not set.`
No crash.

- [ ] **Step 3: Commit**

```bash
git add convex/init.ts
git commit -m "chore: make init.ts a no-op when STRIPE_SECRET_KEY is absent"
```

---

## Task 7: Clean up http.ts (remove subscription webhooks)

**Files:**
- Modify: `convex/http.ts`

The webhook handler for `customer.subscription.updated` and `customer.subscription.deleted` references types that will eventually be removed. Replace with a minimal placeholder — the Stripe webhook handler will be rebuilt in Plan 3.

- [ ] **Step 1: Replace convex/http.ts**

```ts
import { httpRouter } from "convex/server";
import { auth } from "./auth";
import { httpAction } from "@cvx/_generated/server";
import { STRIPE_WEBHOOK_SECRET } from "@cvx/env";
import { stripe } from "@cvx/stripe";
import { ERRORS } from "~/errors";

const http = httpRouter();

/**
 * Gets and constructs a Stripe event signature.
 * Kept here for Plan 3 — will be wired up with Payment Link webhooks.
 */
async function getStripeEvent(request: Request) {
  if (!STRIPE_WEBHOOK_SECRET) {
    throw new Error(`Stripe - ${ERRORS.ENVS_NOT_INITIALIZED}`);
  }
  const signature = request.headers.get("Stripe-Signature");
  if (!signature) throw new Error(ERRORS.STRIPE_MISSING_SIGNATURE);
  const payload = await request.text();
  return stripe.webhooks.constructEventAsync(
    payload,
    signature,
    STRIPE_WEBHOOK_SECRET,
  );
}

// Stripe webhook — will be populated in Plan 3.
http.route({
  path: "/stripe/webhook",
  method: "POST",
  handler: httpAction(async (_ctx, request) => {
    const event = await getStripeEvent(request);
    console.info(`Stripe event received: ${event.type} (handler pending Plan 3)`);
    return new Response(null, { status: 200 });
  }),
});

auth.addHttpRoutes(http);

export default http;
```

- [ ] **Step 2: Clean up convex/stripe.ts**

The current `stripe.ts` exports subscription functions (`PREAUTH_createSubscription`, `PREAUTH_replaceSubscription`, `PREAUTH_deleteSubscription`, `createSubscriptionCheckout`, `createCustomerPortal`, `cancelCurrentUserSubscriptions`, etc.) that reference the now-unused subscription flow and will cause type errors.

**Remove all subscription-related functions.** The `plans` and `subscriptions` schema tables are intentionally retained (to avoid a migration error on the live dev deployment), but no code should reference them after this step. Strip `stripe.ts` to the Stripe client only — all new Stripe functionality (Payment Links) will be added in Plan 3.

Replace `convex/stripe.ts` with:

```ts
import Stripe from "stripe";
import { STRIPE_SECRET_KEY } from "@cvx/env";

export const stripe = new Stripe(STRIPE_SECRET_KEY || "", {
  apiVersion: "2024-06-20",
  typescript: true,
});
```

- [ ] **Step 3: Run typecheck**

```bash
npm run typecheck
```

Expected: zero errors in `convex/` directory. Fix any remaining references to subscription types.

- [ ] **Step 4: Commit**

```bash
git add convex/http.ts convex/stripe.ts
git commit -m "chore: remove subscription webhook handlers and stripe subscription functions"
```

---

## Task 8: Remove i18n

**Files:**
- Delete: `src/i18n.ts`, `public/locales/en/translation.json`, `public/locales/es/translation.json`, `src/ui/language-switcher.tsx`
- Modify: `src/main.tsx`
- Modify: `src/routes/_app/_auth/dashboard/_layout.index.tsx`

- [ ] **Step 1: Delete i18n files**

```bash
rm src/i18n.ts
rm -r public/locales
rm src/ui/language-switcher.tsx
```

- [ ] **Step 2: Update src/main.tsx**

Remove the i18n import and initialization. The file should look like:

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "@tanstack/react-router";
import { router } from "@/router";
import "@/index.css";

const rootElement = document.getElementById("root")!;
if (!rootElement.innerHTML) {
  const root = createRoot(rootElement);
  root.render(
    <StrictMode>
      <RouterProvider router={router} />
    </StrictMode>,
  );
}
```

> Check the actual current `main.tsx` content — it likely has i18n imports and a `<Suspense>` wrapper for translations. Remove those. Keep the ConvexProvider and QueryClient setup intact.

- [ ] **Step 3: Update dashboard index to remove useTranslation**

In `src/routes/_app/_auth/dashboard/_layout.index.tsx`, remove:
- `import { useTranslation } from "react-i18next";`
- `const { t } = useTranslation();`
- Replace `{t("title")}` and `{t("description")}` with plain strings (will be replaced entirely in Plan 2 anyway):
  - `{t("title")}` → `"Your Research Projects"`
  - `{t("description")}` → `"Your research projects will appear here."`

- [ ] **Step 4: Run typecheck**

```bash
npm run typecheck
```

Fix any remaining `useTranslation` or `i18next` references.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: remove i18n (react-i18next, locales, language switcher)"
```

---

## Task 9: Remove onboarding route

**Files:**
- Delete: `src/routes/_app/_auth/onboarding/_layout.tsx`
- Delete: `src/routes/_app/_auth/onboarding/_layout.username.tsx`
- Modify: `src/routes/_app/login/_layout.index.tsx`

Clients go directly to `/dashboard` after login (no username step).

- [ ] **Step 1: Delete onboarding files**

```bash
rm src/routes/_app/_auth/onboarding/_layout.tsx
rm src/routes/_app/_auth/onboarding/_layout.username.tsx
```

- [ ] **Step 2: Update login page to remove onboarding redirect**

In `src/routes/_app/login/_layout.index.tsx`:

1. Remove `import { Route as OnboardingUsernameRoute }` line
2. Replace the `useEffect` redirect logic with role-based routing:

```tsx
useEffect(() => {
  if (isLoading || !isAuthenticated || !user) return;
  // All authenticated users go to /dashboard.
  // The dashboard layout handles role-specific redirects.
  navigate({ to: "/dashboard" });
}, [user, isLoading, isAuthenticated]);
```

3. Remove the GitHub sign-in button and the "Or continue with" divider section (lines 130–156 in the original)

- [ ] **Step 3: Remove language switcher from navigation**

In `src/routes/_app/_auth/dashboard/-ui.navigation.tsx`:
- Remove `import { LanguageSwitcher }` line
- Remove the `<DropdownMenuItem>` block containing `<LanguageSwitcher />`
- Remove the plan badge from the user display (the `<span>` showing "Free" / "Pro")
- Remove the "Upgrade to PRO" dropdown section

- [ ] **Step 4: Run the TanStack Router code generator to update routeTree**

```bash
npm run dev:frontend
```

TanStack Router auto-generates `src/routeTree.gen.ts` when the dev server starts. The deleted routes should disappear from the tree. Stop the server once generation completes (`Ctrl+C`).

- [ ] **Step 5: Run typecheck**

```bash
npm run typecheck
```

Fix references to deleted routes.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: remove onboarding route and GitHub login"
```

---

## Task 10: Add role-based route guards

**Files:**
- Modify: `src/routes/_app/_auth.tsx`
- Create: `src/routes/_app/_auth/_admin.tsx`
- Create: `src/routes/_app/_auth/dashboard/_layout.pending.tsx`

- [ ] **Step 1: Create the "access pending" page**

Create `src/routes/_app/_auth/dashboard/_layout.pending.tsx`:

```tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute(
  "/_app/_auth/dashboard/_layout/pending",
)({
  component: AccessPending,
  beforeLoad: () => ({ title: "Access Pending" }),
});

function AccessPending() {
  return (
    <div className="flex h-full w-full items-center justify-center bg-secondary px-6 py-8 dark:bg-black">
      <div className="flex max-w-md flex-col items-center gap-4 rounded-lg border border-border bg-card p-10 text-center">
        <h2 className="text-xl font-medium text-primary">Access Pending</h2>
        <p className="text-sm text-primary/60">
          Your account is awaiting access. Your consultant will send you an
          invitation shortly. Check your email for next steps.
        </p>
        <p className="text-xs text-primary/40">
          If you believe this is an error, contact your consultant directly.
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Create the admin layout guard**

Create `src/routes/_app/_auth/_admin.tsx`:

```tsx
import { createFileRoute, Outlet, useNavigate } from "@tanstack/react-router";
import { useQuery } from "@tanstack/react-query";
import { convexQuery } from "@convex-dev/react-query";
import { api } from "~/convex/_generated/api";
import { useEffect } from "react";

export const Route = createFileRoute("/_app/_auth/_admin")({
  component: AdminLayout,
});

function AdminLayout() {
  const { data: user, isLoading } = useQuery(
    convexQuery(api.app.getCurrentUser, {}),
  );
  const navigate = useNavigate();

  useEffect(() => {
    if (isLoading) return;
    if (!user || user.role !== "admin") {
      navigate({ to: "/dashboard" });
    }
  }, [user, isLoading]);

  if (isLoading || !user || user.role !== "admin") return null;

  return <Outlet />;
}
```

- [ ] **Step 3: Update _auth.tsx to handle role-based redirects**

Replace `src/routes/_app/_auth.tsx`:

```tsx
import { useConvexAuth } from "@convex-dev/react-query";
import { createFileRoute, Outlet, useNavigate } from "@tanstack/react-router";
import { useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import { convexQuery } from "@convex-dev/react-query";
import { api } from "~/convex/_generated/api";

export const Route = createFileRoute("/_app/_auth")({
  component: AuthLayout,
});

function AuthLayout() {
  const { isAuthenticated, isLoading: authLoading } = useConvexAuth();
  const { data: user, isLoading: userLoading } = useQuery(
    convexQuery(api.app.getCurrentUser, {}),
  );
  const navigate = useNavigate();

  useEffect(() => {
    if (authLoading || userLoading) return;

    if (!isAuthenticated) {
      navigate({ to: "/login" });
      return;
    }

    // Authenticated but no role — show access pending
    if (user && user.role === undefined) {
      // Only redirect if not already on /dashboard/pending
      // (the pending page is inside the dashboard layout, handled there)
    }
  }, [authLoading, userLoading, isAuthenticated, user]);

  if ((authLoading || userLoading) && !isAuthenticated) return null;

  return <Outlet />;
}
```

> **Note:** The `/dashboard/pending` redirect is handled inside `dashboard/_layout.tsx` in the next step, since the layout has access to the user object. The `_auth.tsx` only handles the unauthenticated redirect.

- [ ] **Step 4: Update dashboard layout to redirect role:undefined to /pending**

In `src/routes/_app/_auth/dashboard/_layout.tsx`, update `DashboardLayout`:

```tsx
import { createFileRoute, Outlet, useNavigate } from "@tanstack/react-router";
import { Navigation } from "./-ui.navigation";
import { Header } from "@/ui/header";
import { useQuery } from "@tanstack/react-query";
import { convexQuery } from "@convex-dev/react-query";
import { api } from "@cvx/_generated/api";
import { useEffect } from "react";

export const Route = createFileRoute("/_app/_auth/dashboard/_layout")({
  component: DashboardLayout,
});

function DashboardLayout() {
  const { data: user, isLoading } = useQuery(convexQuery(api.app.getCurrentUser, {}));
  const navigate = useNavigate();

  useEffect(() => {
    if (isLoading || !user) return;
    if (user.role === undefined) {
      navigate({ to: "/dashboard/pending" });
    }
  }, [user, isLoading]);

  if (!user) return null;
  if (user.role === undefined) return null; // redirect in progress

  return (
    <div className="flex min-h-[100vh] w-full flex-col bg-secondary dark:bg-black">
      <Navigation user={user} />
      <Header />
      <Outlet />
    </div>
  );
}
```

- [ ] **Step 5: Regenerate route tree**

```bash
npm run dev:frontend
```

Stop once `routeTree.gen.ts` updates. Verify the new routes appear in the generated tree.

- [ ] **Step 6: Run typecheck**

```bash
npm run typecheck
```

- [ ] **Step 7: Smoke-test role flow**

1. `npm start`
2. Log in with a fresh email (no invite) → should see "Access Pending" page
3. Manually set `role: "admin"` on your own user in the Convex dashboard (Data → users → patch the row)
4. Refresh → should see the Dashboard normally
5. Verify `/admin/*` routes redirect non-admin users to `/dashboard`

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: add role-based route guards — pending page, admin layout"
```

---

## Task 11: Welcome email for new invites

**Files:**
- Create: `convex/email/templates/welcomeEmail.tsx`
- Modify: `convex/invites.ts`

When an admin creates an invite, the client receives a welcome email with login instructions.

- [ ] **Step 1: Create the email template**

Create `convex/email/templates/welcomeEmail.tsx`:

```tsx
import {
  Body,
  Container,
  Head,
  Heading,
  Html,
  Preview,
  Text,
  Link,
  Section,
} from "@react-email/components";
import * as React from "react";

interface WelcomeEmailProps {
  clientName: string;
  loginUrl: string;
  consultantName?: string;
}

export function WelcomeEmail({
  clientName,
  loginUrl,
  consultantName = "Your consultant",
}: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>You have been invited to the client portal</Preview>
      <Body style={{ backgroundColor: "#f9fafb", fontFamily: "sans-serif" }}>
        <Container style={{ maxWidth: 480, margin: "40px auto", padding: 24 }}>
          <Heading style={{ fontSize: 22, color: "#111" }}>
            Welcome, {clientName}
          </Heading>
          <Text style={{ color: "#444" }}>
            {consultantName} has set up a research portal for you. You can
            access your projects and reports by logging in with your email.
          </Text>
          <Section style={{ margin: "24px 0" }}>
            <Link
              href={loginUrl}
              style={{
                backgroundColor: "#111",
                color: "#fff",
                padding: "10px 20px",
                borderRadius: 6,
                textDecoration: "none",
              }}
            >
              Access your portal →
            </Link>
          </Section>
          <Text style={{ color: "#888", fontSize: 13 }}>
            Click the button above, enter your email address, and we will send
            you a one-time login code. No password needed.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}

export async function sendWelcomeEmail({
  email,
  clientName,
  loginUrl,
}: {
  email: string;
  clientName: string;
  loginUrl: string;
}) {
  const { Resend } = await import("resend");
  const { AUTH_RESEND_KEY, AUTH_EMAIL } = await import("@cvx/env");

  const resend = new Resend(AUTH_RESEND_KEY);
  const { error } = await resend.emails.send({
    from: AUTH_EMAIL ?? "Research Portal <onboarding@resend.dev>",
    to: [email],
    subject: "You've been invited to the research portal",
    react: WelcomeEmail({ clientName, loginUrl }),
  });
  if (error) throw new Error(JSON.stringify(error));
}
```

- [ ] **Step 2: Call sendWelcomeEmail from createInvite**

In `convex/invites.ts`, update `createInvite` to send the email after upserting:

```ts
// Add this import at the top:
import { SITE_URL } from "@cvx/env";

// At the end of the createInvite handler, after the upsert:
// (Add after the if/else block)
const { sendWelcomeEmail } = await import("@cvx/email/templates/welcomeEmail");
await sendWelcomeEmail({
  email: args.email,
  clientName: args.name,
  loginUrl: `${SITE_URL}/login`,
});
```

> **Note:** Email sending in Convex mutations requires Resend credentials. In dev, you can temporarily comment this out and test the invite flow without email. Set `AUTH_RESEND_KEY` and `AUTH_EMAIL` in the Convex dashboard (Settings → Environment Variables) when ready.

- [ ] **Step 3: Run typecheck**

```bash
npm run typecheck
```

- [ ] **Step 4: Commit**

```bash
git add convex/email/templates/welcomeEmail.tsx convex/invites.ts
git commit -m "feat: send welcome email when admin creates a client invite"
```

---

## Task 12: Update navigation (remove subscription UI)

**Files:**
- Modify: `src/routes/_app/_auth/dashboard/-ui.navigation.tsx`
- Modify: `src/routes/_app/_auth/dashboard/_layout.settings.billing.tsx`

- [ ] **Step 1: Rewrite navigation**

Replace `src/routes/_app/_auth/dashboard/-ui.navigation.tsx` with a cleaned-up version:
- Remove all plan/subscription references
- Remove language switcher
- Show "Admin" nav link only if `user.role === "admin"`

```tsx
import { Slash, Settings, LogOut } from "lucide-react";
import { cn, useSignOut } from "@/utils/misc";
import { ThemeSwitcher } from "@/ui/theme-switcher";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuLabel,
  DropdownMenuItem,
  DropdownMenuTrigger,
  DropdownMenuSeparator,
} from "@/ui/dropdown-menu";
import { Button } from "@/ui/button";
import { buttonVariants } from "@/ui/button-util";
import { Logo } from "@/ui/logo";
import { Link, useMatchRoute, useNavigate } from "@tanstack/react-router";
import { User } from "~/types";

export function Navigation({ user }: { user: User }) {
  const signOut = useSignOut();
  const matchRoute = useMatchRoute();
  const navigate = useNavigate();
  const isDashboard = matchRoute({ to: "/dashboard" });
  const isSettings = matchRoute({ to: "/dashboard/settings" });
  const isAdmin = matchRoute({ to: "/admin", fuzzy: true });

  if (!user) return null;

  return (
    <nav className="sticky top-0 z-50 flex w-full flex-col border-b border-border bg-card px-6">
      <div className="mx-auto flex w-full max-w-screen-xl items-center justify-between py-3">
        <div className="flex h-10 items-center gap-2">
          <Link to="/dashboard" className="flex h-10 items-center gap-1">
            <Logo />
          </Link>
          <Slash className="h-6 w-6 -rotate-12 stroke-[1.5px] text-primary/10" />
          <DropdownMenu modal={false}>
            <DropdownMenuTrigger asChild>
              <Button
                variant="ghost"
                className="gap-2 px-2 data-[state=open]:bg-primary/5"
              >
                <div className="flex items-center gap-2">
                  {user.avatarUrl ? (
                    <img
                      className="h-8 w-8 rounded-full object-cover"
                      alt={user.name ?? user.email}
                      src={user.avatarUrl}
                    />
                  ) : (
                    <span className="h-8 w-8 rounded-full bg-gradient-to-br from-lime-400 from-10% via-cyan-300 to-blue-500" />
                  )}
                  <p className="text-sm font-medium text-primary/80">
                    {user.name || user.email || ""}
                  </p>
                </div>
              </Button>
            </DropdownMenuTrigger>
            <DropdownMenuContent sideOffset={8} className="min-w-56 bg-card p-2">
              <DropdownMenuLabel className="flex items-center text-xs font-normal text-primary/60">
                {user.email}
              </DropdownMenuLabel>
              <DropdownMenuSeparator className="mx-0 my-2" />
              <DropdownMenuItem
                className="group h-9 w-full cursor-pointer justify-between rounded-md px-2"
                onClick={() => navigate({ to: "/dashboard/settings" })}
              >
                <span className="text-sm text-primary/60 group-hover:text-primary">
                  Settings
                </span>
                <Settings className="h-[18px] w-[18px] stroke-[1.5px] text-primary/60" />
              </DropdownMenuItem>
              <DropdownMenuItem
                className="group flex h-9 justify-between rounded-md px-2 hover:bg-transparent"
              >
                <span className="w-full text-sm text-primary/60 group-hover:text-primary">
                  Theme
                </span>
                <ThemeSwitcher />
              </DropdownMenuItem>
              <DropdownMenuSeparator className="mx-0 my-2" />
              <DropdownMenuItem
                className="group h-9 w-full cursor-pointer justify-between rounded-md px-2"
                onClick={() => signOut()}
              >
                <span className="text-sm text-primary/60 group-hover:text-primary">
                  Log Out
                </span>
                <LogOut className="h-[18px] w-[18px] stroke-[1.5px] text-primary/60" />
              </DropdownMenuItem>
            </DropdownMenuContent>
          </DropdownMenu>
        </div>
      </div>

      {/* Tab bar */}
      <div className="mx-auto flex w-full max-w-screen-xl items-center gap-3">
        <div className={cn("flex h-12 items-center border-b-2", isDashboard ? "border-primary" : "border-transparent")}>
          <Link to="/dashboard" className={cn(`${buttonVariants({ variant: "ghost", size: "sm" })} text-primary/80`)}>
            Projects
          </Link>
        </div>
        <div className={cn("flex h-12 items-center border-b-2", isSettings ? "border-primary" : "border-transparent")}>
          <Link to="/dashboard/settings" className={cn(`${buttonVariants({ variant: "ghost", size: "sm" })} text-primary/80`)}>
            Settings
          </Link>
        </div>
        {user.role === "admin" && (
          <div className={cn("flex h-12 items-center border-b-2", isAdmin ? "border-primary" : "border-transparent")}>
            <Link to="/admin" className={cn(`${buttonVariants({ variant: "ghost", size: "sm" })} text-primary/80`)}>
              Admin
            </Link>
          </div>
        )}
      </div>
    </nav>
  );
}
```

- [ ] **Step 2: Replace billing settings with a placeholder**

Replace `src/routes/_app/_auth/dashboard/_layout.settings.billing.tsx`:

```tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute(
  "/_app/_auth/dashboard/_layout/settings/billing",
)({
  component: BillingSettings,
  beforeLoad: () => ({
    title: "Billing",
    headerTitle: "Billing",
    headerDescription: "View your project invoices.",
  }),
});

function BillingSettings() {
  return (
    <div className="flex h-full w-full flex-col gap-6 p-6">
      <div className="rounded-lg border border-border bg-card p-6">
        <h2 className="text-xl font-medium text-primary">Invoices</h2>
        <p className="mt-1 text-sm text-primary/60">
          Your project invoices will appear here. (Coming in Plan 3)
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Run typecheck and fix remaining errors**

```bash
npm run typecheck
```

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor: update navigation — remove subscription UI, add admin tab"
```

---

## Task 13: Final verification

- [ ] **Step 1: Run all tests**

```bash
npx vitest run
```

Expected: All tests pass.

- [ ] **Step 2: Run full typecheck**

```bash
npm run typecheck
```

Expected: Zero errors.

- [ ] **Step 3: Run linter**

```bash
npm run lint
```

Fix any lint errors.

- [ ] **Step 4: Start full dev server and smoke-test**

```bash
npm start
```

Verify:
1. `/` — loads (template home page, will be replaced in Plan 3)
2. `/login` — shows email form only (no GitHub button)
3. Log in with your email → lands on `/dashboard`
4. Dashboard shows "Your Research Projects" placeholder
5. Admin tab visible if your user has `role: "admin"`
6. New email (no invite) → "Access Pending" page
7. Convex dev server shows no errors

- [ ] **Step 5: Final commit**

```bash
git add -A
git commit -m "feat: Plan 1 complete — foundation, schema, invite flow, role-based auth"
```

---

## What's Next

**Plan 2: Portal** — builds on this foundation:
- Admin panel: `/admin/clients`, `/admin/projects`, `/admin/reports/:id`, `/admin/billing`
- Client portal: project cards dashboard, tabbed report viewer, PDF download
- Convex queries/mutations for projects and reports

**Plan 3: Payments + Landing**
- Stripe Payment Link generation
- `checkout.session.completed` webhook handler
- Public landing page (`/`)
