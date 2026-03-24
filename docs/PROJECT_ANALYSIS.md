# Project Analysis

## Executive Summary

This repository is a full-stack finance tracking application with a fairly clean separation between UI, server actions, persistence, and background jobs. The design is centered on a Next.js App Router frontend backed by Prisma/PostgreSQL, with Clerk handling authentication and Inngest handling recurring or scheduled operations.

At a product level, the app combines four main concerns:

- Personal account management
- Transaction tracking
- Budget monitoring
- AI-assisted receipt ingestion and reporting

## How The App Is Organized

### Routing

The `app/` directory uses route groups:

- `(auth)` contains Clerk sign-in and sign-up pages
- `(main)` contains authenticated product pages
- `api/` exposes the Inngest handler and a seed route

This keeps public auth pages separate from the main product UI while preserving a shared root layout.

### Layouts and shared shell

`app/layout.js` provides:

- `ClerkProvider`
- the shared header
- the global toaster
- the site footer

`components/header.jsx` acts as both navigation and a lightweight onboarding bridge by calling `checkUser()` to synchronize the Clerk identity into Prisma.

### Server-side actions

Most business logic lives in `actions/`:

- `dashboard.js`: account creation, account listing, dashboard transaction feed
- `account.js`: account detail retrieval, bulk transaction deletion, default account switching
- `transaction.js`: create, read, update, and AI receipt scanning
- `budget.js`: monthly budget read/update
- `send-email.js`: Resend wrapper
- `seed.js`: demo transaction generation

This makes the app feel close to a thin-client architecture: UI components collect input, while trusted operations happen on the server.

## Data Model Review

### User

The `User` model is the local application identity. It stores the Clerk user ID plus profile metadata and owns all finance records.

### Account

Accounts represent the main financial containers. Each account has:

- a type (`CURRENT` or `SAVINGS`)
- a balance
- a default flag
- a one-to-many relationship with transactions

The code assumes there should always be one meaningful default account for dashboard budgeting.

### Transaction

Transactions are the operational core of the app. They capture:

- type (`INCOME` or `EXPENSE`)
- amount
- description
- category
- date
- recurrence state
- completion status

Recurring metadata is stored directly on the transaction row, which makes scheduling straightforward but also means transaction updates need to be careful about account balances.

### Budget

There is one budget per user rather than one budget per account. The UI applies that budget to the default account's monthly expenses.

This is a valid simplification, but it is worth noting that the schema and the UX are not truly multi-budget yet.

## Main Runtime Flows

### 1. Sign-in to local user creation

When the header renders for a signed-in user, `checkUser()`:

1. Reads the Clerk user
2. Looks up the matching Prisma user by `clerkUserId`
3. Creates a new Prisma user if missing

This keeps downstream finance logic working with local relational IDs.

### 2. Creating an account

`createAccount()`:

1. Verifies auth
2. Applies Arcjet rate limiting
3. Resolves the local user
4. Parses the starting balance
5. Forces the first account to become default
6. Clears any previous default when needed
7. Writes the account

This is a sensible flow and keeps the "default account" invariant in one place.

### 3. Creating a transaction

`createTransaction()`:

1. Verifies auth
2. Applies Arcjet protection
3. Resolves the local user
4. Verifies the selected account
5. Calculates the account balance effect
6. Creates the transaction inside a DB transaction
7. Updates the account balance in the same DB transaction

This is the most important consistency path in the app.

### 4. Editing a transaction

Transaction editing is more subtle than creation because it must reconcile the old transaction effect with the new one. The current implementation now correctly handles:

- amount changes
- type changes
- account changes

Without this logic, balances drift over time.

### 5. Receipt scanning

The receipt flow is:

1. User uploads image
2. Client sends the file to the server action
3. Server converts it to base64
4. Gemini receives both the image and a JSON-only extraction prompt
5. Response is parsed and mapped back into the form

This is a good practical AI pattern: limited output shape, clear category constraints, and form prefilling instead of fully autonomous writes.

### 6. Budget alerts and monthly reports

Inngest runs scheduled jobs to:

- trigger recurring transactions daily
- generate monthly reports on the first day of the month
- check budget usage every 6 hours

The monthly report path also uses Gemini for insight generation, which gives the app a second AI surface beyond OCR-like extraction.

## Security and Protection

The project uses multiple layers:

- Clerk middleware protects authenticated product routes
- Arcjet middleware adds shield and bot detection
- Arcjet token bucket rate limiting protects sensitive server actions
- Prisma queries are scoped to the authenticated user

This is a solid baseline for a side project or portfolio-grade SaaS starter.

## Important Operational Dependencies

The repo is not standalone. To run end to end, it needs:

- PostgreSQL
- Clerk
- Gemini API access
- Resend
- Arcjet
- Inngest runtime/dev server for scheduled jobs

Because of that, the project should be documented as an integration-heavy app, not just a plain Next.js sample.

## Current Strengths

- Good stack selection for a modern SaaS-style app
- Clear route grouping
- Business logic mostly centralized in server actions
- Useful feature set for a demo or starter
- Sensible Prisma schema for the current scope
- Real AI feature integration with concrete product value

## Current Constraints and Risks

### Placeholder environment values

The checked-in `.env.example` uses placeholder Clerk keys. If those same placeholder values are copied into `.env`, `next build` fails during route configuration because Clerk expects real key formats.

### Seed route is not generic

`actions/seed.js` depends on hardcoded `ACCOUNT_ID` and `USER_ID`, so `/api/seed` is only safe after manually adapting those constants.

### Budget model is user-level, not account-level

The UI presents budget progress for the default account, but the schema only stores one budget per user. That is okay for now, but it limits future budgeting features.

### Background jobs depend on service wiring

The Inngest functions are implemented, but real automation depends on the runtime being active and connected correctly in each environment.

## Suggested Next Improvements

- Add account-level or category-level budgets
- Replace the seed route with a user-aware seeded onboarding flow
- Add tests for balance reconciliation when editing and deleting transactions
- Add explicit onboarding or change Clerk redirect URLs if onboarding is not planned
- Add a deployment guide for Vercel plus required third-party services
- Add stronger validation around AI receipt extraction fallback cases

## Summary

This is a serious full-stack starter rather than a simple demo page. It already covers auth, protected routes, persistence, background automation, transactional balance updates, email delivery, and AI-assisted extraction. With real credentials and a few hardening steps, it is a strong private portfolio project or a foundation for a finance SaaS MVP.
