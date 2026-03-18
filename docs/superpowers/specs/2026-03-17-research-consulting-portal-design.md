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
Project cards grid. Each card shows project title, status badge (color-coded), and estimated delivery date. Statuses: `Awaiting Payment` (indigo) | `In Progress` (amber) | `Delivered` (green).

**Report View (`/dashboard/projects/:id`)**
Tabbed layout with five tabs: Summary · Key Findings · Data & Charts · Recommendations · Methodology. PDF download button persistent in the header. Report sections are composed by the consultant in the admin panel and rendered as rich content blocks (text, stat callouts, chart data).

**Billing (`/dashboard/billing`)**
List of projects with payment status and invoice links for past transactions.

**Account Settings (`/dashboard/settings`)**
Profile info, email preferences.

### Admin Panel (authenticated, role: `admin`)

Accessible only to the consultant's account. Separate nav section within the dashboard.

**Clients (`/admin/clients`)**
Create client accounts, send invite emails, view/impersonate client portal.

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
| `status` | `"awaiting_payment" \| "in_progress" \| "delivered"` | |
| `price` | `number` | In cents |
| `stripePaymentLinkId` | `string?` | Stripe Payment Link ID |
| `stripePaymentIntentId` | `string?` | Set after payment confirmed |
| `estimatedDelivery` | `number?` | Unix timestamp |
| `createdAt` | `number` | |

### `reports`
| Field | Type | Notes |
|---|---|---|
| `projectId` | `Id<"projects">` | |
| `pdfStorageId` | `Id<"_storage">?` | Convex file storage |
| `sections` | `ReportSection[]` | JSON array of tab content |
| `published` | `boolean` | False until consultant publishes |

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
Stored on the `users` table as a `role` field: `"client" | "admin"`. The admin role is assigned manually in the Convex dashboard on the consultant's own account.

---

## Payment Flow

1. Consultant creates a project in Admin → sets price
2. Consultant clicks "Generate Payment Link" → Convex action calls Stripe API to create a Payment Link (one-time charge)
3. Project status set to `awaiting_payment`, Payment Link stored on project
4. Client sees project card with "Pay Now" button → opens Stripe-hosted checkout
5. Stripe fires `checkout.session.completed` webhook → Convex HTTP action receives it → project status flips to `in_progress` automatically
6. Client receives email: "Your project is now in progress"
7. Consultant completes research, uploads report in Admin, publishes it → status flips to `delivered`
8. Client receives email: "Your report is ready"

---

## Auth & Access Control

- All `/dashboard/*` and `/admin/*` routes require authentication (handled by TanStack Router guards, existing in template)
- Admin routes additionally check `user.role === "admin"` — redirect to dashboard if not admin
- Report data is fetched with a Convex query that checks `userId` matches the authenticated user (clients can only see their own projects)
- Admin queries bypass the userId filter

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

- `projects` and `reports` tables with Convex schema
- Admin panel routes and UI
- Client invite flow (email with magic link)
- Report builder UI (admin side)
- Tabbed report viewer (client side)
- PDF upload + download via Convex file storage
- Stripe Payment Link generation (one-time, not subscription)
- Role-based route guards (`admin` vs `client`)
- Landing page (replacing the template's placeholder home)

---

## Out of Scope (for now)

- Self-serve signup (clients cannot create their own accounts)
- E-signature / contract signing (clickthrough ToS at login is sufficient)
- Multiple consultants / team accounts
- Report comments or client feedback
- Automated billing / recurring retainers
