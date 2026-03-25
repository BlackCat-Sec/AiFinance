# AI Finance Platform

Full-stack personal finance platform built with Next.js App Router, Clerk, Prisma, PostgreSQL, Arcjet, Gemini, Resend, and Inngest. The app lets authenticated users create accounts, track income and expenses, scan receipts with AI, set monthly budgets, and automate recurring transaction processing plus email-based reporting.

## What The App Does

- Authenticates users with Clerk and auto-provisions a local `User` record on first sign-in.
- Lets each user manage multiple accounts with one default account.
- Supports manual income and expense tracking with categories and recurring schedules.
- Uses Gemini to extract amount, date, merchant, description, and category from uploaded receipt images.
- Shows dashboard summaries, recent transactions, account-level charts, and budget progress.
- Sends monthly financial reports and budget alerts through Resend.
- Runs background automation with Inngest for recurring transactions, budget checks, and monthly reports.
- Protects routes and server actions with Clerk and Arcjet.

## Tech Stack

- `Next.js 15` with the App Router
- `React 19 RC`
- `Prisma 6` with PostgreSQL
- `Clerk` for auth
- `Arcjet` for middleware protection and rate limiting
- `Inngest` for scheduled/background jobs
- `Gemini` via `@google/generative-ai`
- `Resend` + React Email for outbound email
- `Tailwind CSS`, Radix UI, shadcn-style components, Recharts

## Tooling Breakdown

The project depends on several external tools and libraries. The important point for a user or maintainer is not just what they are, but what responsibility each one owns in this codebase.

| Tool | How it is used here | Why it is used |
| --- | --- | --- |
| `Next.js` | App Router pages, layouts, API routes, and server actions | Keeps frontend and backend logic in one framework |
| `React` | UI components, client-side forms, charts, and interactions | Standard component model for the app UI |
| `Clerk` | Sign-in/sign-up UI, session handling, protected routes, user identity | Offloads authentication and user session management |
| `Prisma` | Database queries, schema definition, migrations, client generation | Centralizes data access and keeps schema changes manageable |
| `PostgreSQL` | Persistent storage for users, accounts, budgets, and transactions | Relational data fits the financial records model well |
| `Arcjet` | Middleware protection, bot detection, and rate limiting on sensitive actions | Reduces abuse and protects public-facing routes/actions |
| `Gemini` | Receipt parsing and monthly insight generation | Handles the two AI-specific parts of the product |
| `Inngest` | Cron-like scheduled jobs and recurring background workflows | Keeps recurring transactions, alerts, and reports out of request-time logic |
| `Resend` | Outbound email delivery | Sends reports and budget alerts reliably |
| `React Email` | Email template rendering | Keeps email content versioned in code as React components |
| `Tailwind CSS` | Layout, spacing, colors, utility styling | Fast UI iteration without a large custom CSS system |
| `Radix UI` | Accessible primitives such as dialog, select, popover, switch | Provides stable low-level UI building blocks |
| `Recharts` | Charts on dashboard/account pages | Covers the financial visualization layer |
| `React Hook Form` + `Zod` | Transaction/account form state and validation | Keeps form validation explicit and predictable |
| `Sonner` | Toast notifications | Lightweight feedback for create/update actions |

For a longer tool-by-tool explanation, see [`docs/TOOLS_AND_SERVICES.md`](docs/TOOLS_AND_SERVICES.md).

## Project Structure

```text
app/
  (auth)/                 Clerk sign-in/sign-up pages
  (main)/                 Authenticated product UI
    account/[id]/         Account details, chart, and transaction table
    dashboard/            Account cards, budget progress, transaction overview
    transaction/create/   Create/edit transaction flow
  api/inngest/            Inngest handler endpoint
  api/seed/               Demo seeding endpoint
actions/                  Server actions for accounts, budget, dashboard, email, seed, transactions
components/               Shared UI and layout components
data/                     Landing page and category definitions
emails/                   React Email template
hooks/                    Client helpers such as useFetch
lib/                      Prisma client, Arcjet config, Clerk user sync, Inngest client/functions
prisma/                   Prisma schema and migrations
```

## Core Flows

### Authentication and user sync

`components/header.jsx` calls `checkUser()` in `lib/checkUser.js`. When a signed-in Clerk user hits the app, the code ensures a matching Prisma `User` row exists.

### Account management

`actions/dashboard.js` handles account creation and dashboard account queries. The first created account is forced to be the default account. Users can later switch the default account from the dashboard.

### Transaction lifecycle

`actions/transaction.js` creates, fetches, updates, and scans transactions. Creating or editing a transaction also updates the related account balance so the account total stays in sync with the ledger.

### Receipt scanning

`app/(main)/transaction/_components/recipt-scanner.jsx` uploads an image file to the `scanReceipt` server action. Gemini is prompted to return strictly formatted JSON so the form can auto-fill the transaction values.

### Budgeting

`actions/budget.js` stores one monthly budget per user. The dashboard compares current-month expense totals against that value for the default account.

### Background jobs

`lib/inngest/function.js` defines three automations:

- Daily recurring transaction triggering
- Monthly financial reports
- Budget alert checks every 6 hours

## Database Model

The Prisma schema defines four core models:

- `User`: local app user mapped to a Clerk user ID
- `Account`: current or savings account with a live balance and default flag
- `Transaction`: income/expense entry with recurrence metadata and status
- `Budget`: one monthly budget per user, plus last alert timestamp

## Environment Variables

Create `.env` from `.env.example` and provide real values for:

- `DATABASE_URL`
- `DIRECT_URL`
- `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- `CLERK_SECRET_KEY`
- `NEXT_PUBLIC_CLERK_SIGN_IN_URL`
- `NEXT_PUBLIC_CLERK_SIGN_UP_URL`
- `NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL`
- `NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL`
- `GEMINI_API_KEY`
- `RESEND_API_KEY`
- `ARCJET_KEY`

Important:

- The sample Clerk keys in `.env.example` are placeholders only.
- A production build will fail if placeholder Clerk values like `pk_test_...` or `sk_test_...` are left in `.env`.
- The redirect URLs in the example env point to `/dashboard`, which matches the routes present in this repo.

## Local Setup

1. Install dependencies:

   ```powershell
   npm install
   ```

2. Copy the env template:

   ```powershell
   Copy-Item .env.example .env
   ```

3. Add real service keys to `.env`.

4. Run Prisma migrations:

   ```powershell
   npx prisma migrate dev
   ```

5. Start the dev server:

   ```powershell
   npm run dev
   ```

6. Optional: start Inngest locally if you want scheduled jobs running during development.

## Verification Notes

- `npm run lint` succeeds with the current source.
- `npm run build` requires valid Clerk environment values. Placeholder keys from `.env.example` will cause build-time failure.

## Known Constraints

- `/api/seed` depends on hardcoded `ACCOUNT_ID` and `USER_ID` values in `actions/seed.js`.
- The app assumes external services are configured: Clerk, PostgreSQL, Gemini, Resend, and Arcjet.
- Background jobs are defined, but local scheduling depends on the Inngest dev/runtime setup.

## Extra Documentation

See [`docs/PROJECT_ANALYSIS.md`](docs/PROJECT_ANALYSIS.md) for a deeper architecture walkthrough and implementation notes.

See [`docs/TOOLS_AND_SERVICES.md`](docs/TOOLS_AND_SERVICES.md) for a detailed explanation of each major tool, where it appears in the codebase, why it exists, and what a user needs to know before configuring it.

---
Reference tutorial inspiration: https://youtu.be/egS6fnZAdzk
