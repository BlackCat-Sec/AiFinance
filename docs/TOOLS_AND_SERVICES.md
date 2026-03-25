# Tools and Services

## Purpose of This Document

This document explains the main tools, services, and libraries used in this project. For each one, it answers four questions:

- What is it
- Where is it used in this project
- Why is it used
- What should a user or maintainer know before working with it

No secrets, tokens, or private configuration values are documented here.

## Core Framework

### Next.js

**What it is**

Next.js is the full-stack web framework that hosts the application.

**How it is used here**

- App Router pages live under `app/`
- Shared layouts are defined in `app/layout.js` and route-group layouts
- API handlers exist under `app/api/`
- Server actions are called from UI components
- The build, dev, and start lifecycle all run through Next.js

**Why it is used**

- It supports server-rendered and client-rendered UI in one codebase
- It simplifies routing and page composition
- It works well with server actions for data mutations
- It allows API endpoints and UI to live together

**What users should know**

- Main scripts: `npm run dev`, `npm run build`, `npm run start`
- Production builds depend on valid environment variables, especially auth-related ones
- Route groups like `(auth)` and `(main)` help separate public vs authenticated pages

### React

**What it is**

React is the component library used to build the interface.

**How it is used here**

- UI components in `components/`
- Client-side feature components in `app/(main)/.../_components/`
- Form interactions, chart rendering, and local state management

**Why it is used**

- It provides a clear component model for the dashboard and forms
- It works natively with Next.js

**What users should know**

- Some components are server components by default
- Files marked with `"use client"` run in the browser and can use hooks and local state

## Authentication and Access Control

### Clerk

**What it is**

Clerk is the authentication provider.

**How it is used here**

- `app/(auth)/sign-in/...` and `app/(auth)/sign-up/...` render Clerk auth components
- `ClerkProvider` is mounted in `app/layout.js`
- Middleware uses Clerk to protect `/dashboard`, `/account`, and `/transaction` routes
- `lib/checkUser.js` maps the signed-in Clerk user into the local Prisma `User` table

**Why it is used**

- It removes the need to build password/session flows manually
- It provides a ready-made sign-in/sign-up UI
- It integrates cleanly with Next.js middleware and server code

**What users should know**

- Real Clerk keys are required for a working build
- Placeholder keys will break production build and auth flows
- Redirect URLs in `.env.example` are set to `/dashboard` because that route exists in this app

### Arcjet

**What it is**

Arcjet is a protection and rate-limiting layer.

**How it is used here**

- `middleware.js` applies bot detection and shield protection
- `lib/arcjet.js` configures a token bucket rate limit
- Sensitive server actions such as account creation and transaction creation call Arcjet before writing data

**Why it is used**

- Prevents basic abuse on public or authenticated endpoints
- Adds a consistent rate-limiting mechanism outside the UI layer
- Protects actions that could otherwise be spammed

**What users should know**

- A valid `ARCJET_KEY` is required for runtime protection
- Arcjet is not a replacement for auth; it complements auth

## Database and Persistence

### PostgreSQL

**What it is**

PostgreSQL is the primary database.

**How it is used here**

- Stores users, accounts, transactions, and budgets
- Accessed through Prisma

**Why it is used**

- Financial records are relational data
- Transactions, referential integrity, and structured querying are important here

**What users should know**

- `DATABASE_URL` and `DIRECT_URL` must point to a valid Postgres instance
- Migrations must be applied before the app can work correctly

### Prisma

**What it is**

Prisma is the ORM and schema management tool.

**How it is used here**

- Schema is defined in `prisma/schema.prisma`
- Generated client is imported from `lib/prisma.js`
- Server actions query and mutate the database through Prisma
- Migrations in `prisma/migrations/` track schema evolution

**Why it is used**

- Keeps database access centralized
- Makes relational queries easier to manage
- Provides a structured migration workflow

**What users should know**

- `postinstall` runs `prisma generate`
- Typical setup command: `npx prisma migrate dev`
- If the schema changes, new migrations should be created rather than editing old ones

## AI and Automation

### Gemini

**What it is**

Gemini is the AI model provider used in the app.

**How it is used here**

- `actions/transaction.js` uses it to scan receipt images
- `lib/inngest/function.js` uses it to generate monthly financial insights

**Why it is used**

- The project has two AI-specific jobs:
  - extracting structured receipt data from images
  - generating short narrative insights from monthly stats
- Those jobs are isolated enough to justify a dedicated model integration

**What users should know**

- The app expects a `GEMINI_API_KEY`
- Receipt extraction is prompt-based and not guaranteed to be perfect
- Users should still review AI-filled transaction fields before saving

### Inngest

**What it is**

Inngest is the background job and scheduled workflow system.

**How it is used here**

- `app/api/inngest/route.js` exposes the Inngest handler
- `lib/inngest/function.js` defines recurring transaction processing, monthly reports, and budget alerts

**Why it is used**

- These jobs should not run inside normal page requests
- Scheduling and background retries are a better fit for a workflow engine

**What users should know**

- Local development may need an Inngest dev server/runtime in addition to the Next.js app
- Scheduled flows are part of the product behavior, not optional if full automation is expected

## Email

### Resend

**What it is**

Resend is the email delivery service.

**How it is used here**

- `actions/send-email.js` wraps the provider call
- Used by background workflows to send budget alerts and monthly reports

**Why it is used**

- The app sends operational email, not just transactional database writes
- Email delivery should be delegated to a provider rather than handled directly

**What users should know**

- A valid `RESEND_API_KEY` is required
- Email sending will fail even if the rest of the app works, if this service is not configured

### React Email

**What it is**

React Email is the templating layer for emails.

**How it is used here**

- Email content is built in `emails/template.jsx`
- Resend receives rendered React email content

**Why it is used**

- Keeps email markup versioned inside the codebase
- Easier to maintain than string-built HTML templates

**What users should know**

- `npm run email` starts the local email template dev environment
- Template changes should be tested visually before production use

## UI and Form Libraries

### Tailwind CSS

**What it is**

Tailwind is the utility-first styling system.

**How it is used here**

- Global styles in `app/globals.css`
- Utility classes across nearly all page and component files

**Why it is used**

- Fast iteration on layout and spacing
- Keeps styling close to the components using it

**What users should know**

- Design consistency depends on class reuse rather than a large CSS module system
- Global utility helpers such as gradients are defined in `app/globals.css`

### Radix UI and related UI helpers

**What it is**

Radix provides accessible low-level primitives. The project also uses `vaul` for drawer-style UI and shadcn-style wrappers under `components/ui/`.

**How it is used here**

- Selects, switches, popovers, dialogs, checkboxes, tooltips, and drawers
- Wrapped as reusable UI components in `components/ui/`

**Why it is used**

- Avoids rebuilding basic accessible interaction primitives
- Keeps the component layer consistent

**What users should know**

- Most reusable UI elements are wrappers, not raw library imports
- Project-wide UI updates usually start in `components/ui/`

### React Hook Form

**What it is**

React Hook Form manages form state.

**How it is used here**

- Account creation form
- Transaction creation/edit form

**Why it is used**

- Better control over form state and validation errors
- Good performance with complex form inputs

**What users should know**

- Form logic is mostly in client components
- Validation is paired with Zod

### Zod

**What it is**

Zod is the validation library used for form schemas.

**How it is used here**

- `app/lib/schema.js` defines account and transaction validation rules

**Why it is used**

- Keeps input rules explicit and centralized
- Works directly with React Hook Form through the resolver package

**What users should know**

- If business rules change, update the schema first
- Client validation does not replace server-side data checks

### Recharts

**What it is**

Recharts is the charting library.

**How it is used here**

- Account charts
- Dashboard expense breakdown visuals

**Why it is used**

- The app needs simple financial visualizations rather than a custom chart engine

**What users should know**

- Chart correctness depends on the transaction data being normalized correctly

### Sonner

**What it is**

Sonner is the toast notification library.

**How it is used here**

- Success and error messages during create/update/scan flows

**Why it is used**

- Gives lightweight user feedback without interrupting the page flow

**What users should know**

- Toast usage is mostly tied to the `useFetch` hook and form actions

## Utility Libraries

### date-fns

**What it is**

Date utility library.

**How it is used here**

- Formatting transaction dates
- Monthly calculations
- Seed data generation windows

**Why it is used**

- Safer and clearer than manual date string manipulation

### Lucide React

**What it is**

Icon library.

**How it is used here**

- Buttons, navigation, cards, and charts throughout the UI

**Why it is used**

- Consistent icon set without custom SVG management

## Scripts and Developer Workflow

### `npm run dev`

Starts the Next.js development server.

### `npm run build`

Builds the production bundle. Requires valid environment variables.

### `npm run start`

Runs the built production app after a successful build.

### `npm run lint`

Runs Next.js ESLint checks on the codebase.

### `npm run email`

Starts the React Email development environment for previewing email templates.

### `postinstall`

Runs `prisma generate` automatically after dependency installation.

## Sensitive Information Policy

This repository should never commit:

- real API keys
- real database connection strings
- real service account values
- private user data

Safe documentation includes:

- variable names
- setup instructions
- expected services
- route paths
- architecture descriptions

Unsafe documentation includes:

- actual `.env` contents
- live credentials
- production endpoints that are meant to stay private
- copied secrets from deployment platforms or dashboards
