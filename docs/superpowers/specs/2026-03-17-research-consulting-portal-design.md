# Research Consulting Portal — Design Spec
**Date:** 2026-03-17
**Status:** Approved

---

## Overview

A research consulting services website targeting marketing agencies. The site serves two audiences: a public landing page that establishes credibility and drives inbound inquiries, and a private client portal where agencies access their research projects and reports.

The operator (consultant) manually onboards clients initially, with self-serve discovery as a future goal. The system is built on the existing **convex-saas** template (Convex + TanStack Router + React + Stripe + Tailwind + shadcn), repurposed from a SaaS subscription model into a project-based consulting portal.

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | React + TanStack Router + Tailwind + shadcn |
| Backend | Convex (database, functions, file storage, real-time) |
| Auth | Convex Auth (email code login) |
| Payments | Stripe (one-time Payment Links, not subscriptions) |
| Email | Resend + React Email |
| Hosting | Netlify (frontend) + Convex cloud |

The template's subscription machinery (subscription tiers, billing portal, plan management) is removed or disabled. Stripe is used only for one-time project invoices.

---

## Site Structure

### Public (unauthenticated)

**Landing Page (`/`)**
Credibility-led layout: consultant's photo alongside the headline, value proposition, and service list. Single primary CTA: "Book a Free Call." Sections: Hero → Services → How It Works → About → Contact.

**Login (`/login`)**
Email code authentication (no passwords). Clients receive a one-time code via email.

### Client Portal (authenticated, role: `client`)

**Dashboard (`/dashboard`)**
Project cards grid. Each card shows project title, status badge (color-coded), and estimated delivery date. Statuses: `Awaiting Payment` (indigo) | `In Progress` (amber) | `Delivered` (green). `Cancelled` projects are hidden from the client dashboard entirely — they only appear in the admin project list.

**Report View (`/dashboard/projects/:id`)**
Tabbed layout with five tabs: Summary · Key Findings · Data & Charts · Recommendations · Methodology. PDF download button persistent in the header. Report sections are composed by the consultant in the admin panel and rendered as rich content blocks (text, stat callouts, chart data).

**Billing (`/dashboard/billing`)**
List of projects with payment status and invoice links for past transactions.

**Account Settings (`/dashboard/settings`)**
Profile info, email preferences.

### Admin Panel (authenticated, role: `admin`)

Accessible only to the consultant's account. Separate nav section within the dashboard.

**Clients (`/admin/clients`)**
Create client accounts, send invite emails. Also lists any users with `role: undefined` (arrived without an invite) so the consultant can re-invite them. Preview client portal is deferred.

**Projects (`/admin/projects`)**
Create projects, assign to a client, set price, update status, set estimated delivery date.

**Reports (`/admin/reports/:projectId`)**
Upload PDF (stored in Convex file storage), compose tabbed report sections (rich text + stat callouts + chart data), publish to client.

**Billing (`/admin/billing`)**
View payment status per project. Generate Stripe Payment Link. Manually mark as paid if needed (e.g. wire transfer).

---

## Data Model

Built on top of the template's existing `users`, `authSessions`, and related auth tables.

### `projects`
| Field | Type | Notes |
|---|---|---|
| `userId` | `Id<"users">` | The client this project belongs to |
| `title` | `string` | |
| `description` | `string` | |
| `status` | `"awaiting_payment" \| "in_progress" \| "delivered" \| "cancelled"` | `cancelled` used when consultant cancels a project created in error |
| `price` | `number` | In cents |
| `stripePaymentLinkId` | `string?` | Stripe Payment Link ID |
| `stripePaymentIntentId` | `string?` | Set after payment confirmed via webhook |
| `estimatedDelivery` | `number?` | Unix timestamp |
| `createdAt` | `number` | |

### `reports`
| Field | Type | Notes |
|---|---|---|
| `projectId` | `Id<"projects">` | Join to projects to verify ownership: `reports → projects.userId` |
| `pdfStorageId` | `Id<"_storage">?` | Convex file storage (authenticated URLs, not public) |
| `sections` | `ReportSection[]` | JSON array of tab content |
| `published` | `boolean` | False until consultant publishes |
| `createdAt` | `number` | Unix timestamp |
| `updatedAt` | `number` | Updated on every save |

### `ReportSection` (embedded in report)
```ts
type ReportSection = {
  tab: "summary" | "findings" | "data" | "recommendations" | "methodology";
  blocks: ContentBlock[];
};

type ContentBlock =
  | { type: "text"; content: string }
  | { type: "stat"; label: string; value: string; color?: string }
  | { type: "chart"; chartType: "bar" | "pie" | "line"; data: unknown };
```

### User roles
Stored on the `users` table as a `role` field: `"client" | "admin"`.
- New users created via the admin invite flow receive `role: "client"` automatically (set in the mutation that creates them).
- The consultant's own account receives `role: "admin"`, assigned manually once in the Convex dashboard.

---

## Payment Flow

1. Consultant creates a project in Admin → sets price
2. Consultant clicks "Generate Payment Link" → Convex action calls Stripe API to create a Payment Link with `metadata: { projectId }` and `client_reference_id: projectId`
3. `stripePaymentLinkId` stored on project; status set to `awaiting_payment`
4. Client sees project card with "Pay Now" button → opens Stripe-hosted checkout
5. Stripe fires `checkout.session.completed` webhook → Convex HTTP action verifies signature using `stripe.webhooks.constructEvent(body, sig, STRIPE_WEBHOOK_SECRET)` (same pattern as existing template webhook handler in `convex/stripe.ts`) → reads `projectId` from `session.client_reference_id` (Stripe reliably passes this through from Payment Links; do NOT rely on `metadata` which is not propagated to the session object) → project status flips to `in_progress`, `stripePaymentIntentId` stored from `session.payment_intent`
6. Client receives email: "Your project is now in progress"
7. Consultant completes research, uploads report in Admin, publishes it → status flips to `delivered`
8. Client receives email: "Your report is ready"

### Payment failure / abandonment
If a client abandons checkout or payment is declined, the project remains in `awaiting_payment`. The "Pay Now" button stays visible and links to the same Payment Link. The consultant can regenerate the Payment Link from Admin Billing if needed (e.g. if the link expired).

---

## Auth & Access Control

- All `/dashboard/*` and `/admin/*` routes require authentication (TanStack Router guards, existing in template)
- Admin routes additionally check `user.role === "admin"` via TanStack Router guard — redirect to `/dashboard` if not admin. **This guard is UI-only.** Every admin Convex query and mutation independently re-verifies `user.role === "admin"` server-side. The router guard is a UX convenience, not the security boundary.
- Client-side project queries filter by `userId === currentUser._id` — clients only see their own data
- Report ownership is verified by joining: `reports → projects.userId === currentUser._id`
- Admin queries pass an `asAdmin: true` flag and bypass the userId filter after verifying `user.role === "admin"` server-side

### PDF access
PDF files are served via authenticated URLs generated by `ctx.storage.getUrl(storageId)` inside a Convex query. The URL is fetched **on demand** (when the user clicks "Download PDF"), not embedded in the initial page load, to avoid expiry during long sessions. The query verifies ownership before returning the URL. URLs are not publicly accessible by storage ID alone.

---

## Client Invite Flow

Convex Auth owns the `users` table — rows must be created by the auth system, not by arbitrary mutations. Direct inserts into `users` would orphan the row with no `authAccounts` linkage, causing a duplicate user on first login. The invite flow instead uses a separate `invites` table and Convex Auth's `createOrUpdateUser` hook.

### `invites` table
| Field | Type | Notes |
|---|---|---|
| `email` | `string` | Invited client's email — **indexed, unique constraint enforced by upsert** |
| `name` | `string` | Display name |
| `createdAt` | `number` | |

### Flow
1. Consultant fills in client name and email on `/admin/clients` → submits
2. Convex mutation **upserts** into `invites` by email (replaces any existing row for the same email — handles re-invite / resend). Uses `by_email` index.
3. Resend sends a welcome email directing the client to `/login`
4. Client visits `/login`, enters their email, Convex Auth sends a 6-digit OTP, client enters it
5. On successful OTP verification, Convex Auth calls `createOrUpdateUser` — this callback queries `invites` by email (using `by_email` index). If a matching invite exists: sets `role: "client"` on the new user, stores the name, deletes the invite row
6. Client lands on `/dashboard`
7. If the OTP expires (1 hour), client requests a new one from the login page — no separate resend needed

New users who arrive without a matching invite will be created with `role: undefined`. They see an "access pending" message. The consultant recovers this by re-inviting the same email from `/admin/clients` — this creates a new invite row, and `role: "client"` will be set on the user's next login via the `createOrUpdateUser` hook.

---

## Landing Page Sections

1. **Nav** — Logo, Services, About, Client Login button
2. **Hero** — Consultant photo (right), headline + 1-line value prop (left), "Book a Free Call" CTA
3. **Services** — Consumer Sentiment · Brand Tracking · Competitor Analysis · Audience Profiling
4. **How It Works** — 3-step process: Brief → Research → Portal Delivery
5. **About** — Short bio, credentials, why agencies trust you
6. **Contact / CTA** — Email form or Calendly embed for booking

---

## What We're Removing from the Template

- Subscription plan tiers and plan selection UI
- Stripe subscription webhooks (`customer.subscription.updated`, `customer.subscription.deleted`)
- Subscription billing portal
- i18n / language switcher
- Username onboarding step (clients go straight to dashboard after first login)

---

## What We're Adding

- `projects`, `reports`, and `invites` tables with Convex schema
- Admin panel routes and UI
- Client invite flow (`invites` table + `createOrUpdateUser` hook assigns `role: "client"` on first login)
- Report builder UI (admin side)
- Tabbed report viewer (client side)
- PDF upload + authenticated URL serving via Convex file storage
- Stripe Payment Link generation with `metadata.projectId` for webhook matching
- Stripe webhook signature verification via `stripe.webhooks.constructEvent`
- Role-based route guards (`admin` vs `client`)
- Landing page (replacing the template's placeholder home)

---

## Out of Scope (for now)

- Self-serve signup (clients cannot create their own accounts)
- E-signature / contract signing (clickthrough ToS at login is sufficient)
- Multiple consultants / team accounts
- Report comments or client feedback
- Automated billing / recurring retainers
