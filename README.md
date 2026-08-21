# RentFlow

A property management SaaS for Kenyan landlords: portfolio, leases, M-Pesa rent collection, utility billing, expenses and maintenance in one dashboard, plus a lightweight tenant portal.

**The source code for RentFlow is private.** This page exists so the work can be reviewed without access to the repository — what was built, what it runs on, and which decisions are worth talking about.

**Live:** https://rentflow.zidi.digital

---

## The problem

Kenyan landlords run their portfolios across notebooks, spreadsheets and M-Pesa SMS. Rent arrives on a mobile money rail that no generic property tool speaks to, utility bills are meter-read and hand-calculated, and chasing tenants is manual. RentFlow puts collection, billing and tenant communication in one place, and pushes routine work (payments, maintenance reports) onto a tenant portal so the landlord isn't the middleman.

---

## What it does

**Landlord side**
- Properties, units and leases — creating a lease provisions the tenant account, emails credentials, and auto-generates the monthly rent payment schedule for the full term.
- Payments — list, filters, summary of collected/outstanding/overdue, manual mark-paid, one-off reminders, printable rent invoices. Pending payments past due date are auto-flagged `LATE`.
- Utility billing — meter readings in, total calculated, printable invoice; email to tenant on the paid tier.
- Expense tracking by property and category.
- Maintenance queue with status/priority filters and a written response visible to the tenant.
- Dashboard — occupancy, revenue with month-over-month trend, 6-month expected-vs-collected chart, lease renewal alerts at 60/14 days, and a monthly admin checklist that resets each month.

**Tenant side**
- Pay rent by M-Pesa STK Push from the payment row; the app polls until confirmation and marks it paid.
- View lease, payment history, and utility bills with printable invoices.
- Submit and track maintenance requests.

**Cross-cutting**
- Role-based routing and a session guard on all app routes.
- In-app notifications (lease, payment, maintenance events) with an unread badge.
- Transactional email: tenant welcome, lease agreement, password reset, payment reminders; a daily cron sends automated reminders for paid-tier landlords.

---

## Stack

TypeScript on Bun. Next.js (App Router) with tRPC and TanStack Query, Prisma against PostgreSQL on Neon, Better Auth for sessions, Zod + React Hook Form for validation, HeroUI + TailwindCSS with Recharts and Framer Motion. Safaricom M-Pesa Daraja for payments, Resend for email, UploadThing for files, Biome for linting. Deployed on Vercel with Vercel Cron.

---

## Engineering decisions worth defending

**1. One M-Pesa callback, two payment meanings.** Rent and subscription upgrades both go out as STK Push and both come back to the same webhook. The callback disambiguates them by `checkoutRequestId`, then either marks the rent payment `PAID` or activates the PRO plan. Phone numbers arrive as `07XX`, `7XX` or `2547XX` and are normalised to `2547XX` before the push — one format at the boundary rather than defensive parsing scattered through the code. Daraja credentials are optional: if they're absent, the payment features degrade gracefully instead of breaking the app.

**2. Feature gating drawn along infrastructure cost lines, not marketing lines.** The paid tier gates exactly the things that consume metered third-party quotas — automated cron email reminders and utility-bill emails (Resend), document uploads (storage). Everything a landlord needs to actually run a property stays free: full dashboard, leases, M-Pesa collection, expenses, maintenance, and *printable* utility invoices, which cost nothing to produce. Manual, landlord-triggered reminders are free; automated ones are not, because volume is the cost driver. Database and bandwidth aren't gated because the free tiers there aren't the binding constraint at current scale.

**3. Serverless Postgres has sharp edges, so connection paths are split.** Runtime queries use the pooled Neon connection; schema changes run against the direct one. Running Prisma migrations through the pooler times out on an advisory lock, so that path is deliberately avoided rather than worked around at deploy time. The Prisma/Postgres and native crypto packages are also declared as server-external so they resolve correctly in Vercel's bundling — a small config decision that is the difference between the app running and not.

---

**Live:** https://rentflow.zidi.digital · Source private; happy to walk through the code in an interview.