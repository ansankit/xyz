<div align="center">

# Innovun — Custom Marketing CRM Suite

Modern, type-safe marketing CRM built with a monorepo architecture. Includes a Next.js web app, an Express API, a shared Prisma database package, a UI component library, and shared tooling.

</div>

---

## Tech Stack

- **Monorepo**: Turborepo
- **Language**: TypeScript (Node >= 18)
- **Frontend**: Next.js 15, React 19 (`apps/web`)
- **API**: Express 5 (`apps/api`) on port `4000` by default
- **Database**: PostgreSQL + Prisma ORM (`packages/db`)
  - **Local**: PostgreSQL 15 via Docker (`docker-compose.yml`, port `5433`)
  - **Production**: Supabase Postgres (managed, SSL required)
- **UI Library**: `@repo/ui` shared components
- **Linting/Build**: ESLint 9, TypeScript 5.9, Turbo tasks
- **Formatting**: Prettier 3

## Repository Structure

```
custom-marketing-crm-suite/
  apps/
    api/                  # Express API server
      src/index.ts
    web/                  # Next.js app
      app/
  packages/
    db/                   # Prisma schema, client, seeds, switch scripts
      prisma/
        schema.prisma
        seed.ts
      switch-db.ps1 | switch-db.sh
    ui/                   # Shared UI components
    eslint-config/        # Shared ESLint configs
    typescript-config/    # Shared TS configs
  turbo.json              # Turborepo task pipeline
  docker-compose.yml      # Local Postgres service
  DEVELOPMENT_WORKFLOW.md # Daily env workflow
```

## Local Development

Prerequisites:
- Node 18+
- Docker Desktop (for local PostgreSQL)
- PowerShell (Windows) for `db:switch:*` scripts

1) Install dependencies
```bash
npm install
```

2) Start local Postgres via Docker
```bash
docker-compose up -d
# Exposes Postgres on localhost:5433 with DB=innovun_crm, user=postgres, password=password
```

3) Configure environment
Create a root `.env` with:
```bash
DATABASE_URL="postgresql://postgres:password@localhost:5433/innovun_crm?schema=public"
DIRECT_URL="postgresql://postgres:password@localhost:5433/innovun_crm?schema=public"
```

4) Generate and apply database
```bash
npm run db:generate
npm run db:deploy
npm run db:seed
```

5) Run apps
```bash
# Run all dev servers (un-cached, persistent)
npm run dev

# Or run individually
cd apps/api && npm run dev         # http://localhost:4000
cd apps/web && npm run dev         # http://localhost:3000
```

Useful DB scripts (from repo root):
```bash
npm run db:switch:local       # point db package to local .env
npm run db:switch:production  # point db package to production .env
npm run db:studio             # Prisma Studio
npm run db:reset              # Reset database
```

Note: Prisma CLI now loads environment from the root `.env` via `dotenv-cli` wrapping in `@repo/db` scripts.

### Switch between local and production environments (cross-platform)

Use the root scripts to swap the active `.env`:
```bash
# Use local Docker database
npm run env:switch:local

# Use production database (Supabase)
npm run env:switch:production
```

Create these files at the repo root (not committed):
```bash
# .env.local
DATABASE_URL="postgresql://postgres:password@localhost:5433/innovun_crm?schema=public"
DIRECT_URL="postgresql://postgres:password@localhost:5433/innovun_crm?schema=public"

# .env.production (Supabase example placeholders)
DATABASE_URL="postgresql://USER:PASSWORD@db.<project-ref>.supabase.co:5432/postgres?sslmode=require&schema=public"
DIRECT_URL="postgresql://USER:PASSWORD@db.<project-ref>.supabase.co:5432/postgres?sslmode=require&schema=public"
```

## Common Scripts

From repository root:
- `npm run dev`: run all apps in dev
- `npm run build`: build all apps/packages
- `npm run lint`: lint all packages
- `npm run check-types`: type-check all packages
- `npm run db:*`: proxy Prisma tasks to `@repo/db`

From `apps/web`:
- `npm run dev` (Next.js on port 3000)
- `npm run build`, `npm run start`, `npm run lint`, `npm run check-types`

From `apps/api`:
- `npm run dev` (Express ts-node-dev)
- `npm run build` (tsc), `npm start` (node dist)

From `packages/db`:
- `npm run prisma:generate|migrate|deploy|seed|studio|reset`

## Environment Configuration

- Prisma (via `@repo/db` scripts) now loads variables from the ROOT `.env` using `dotenv-cli`.
  - Required keys: `DATABASE_URL`, `DIRECT_URL`
  - This applies to: `db:generate`, `db:migrate`, `db:deploy`, `db:reset`, `db:studio`, `db:seed`.
- Next.js (`apps/web`) env conventions:
  - Prefix client-exposed vars with `NEXT_PUBLIC_` (e.g., `NEXT_PUBLIC_API_URL`).
  - Per Next.js rules, `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.test` are supported (later files override earlier).
- API (`apps/api`) loads env via `dotenv.config()` from its working directory. For runtime DB access, ensure `DATABASE_URL` is set in the process env or `apps/api/.env.local`.
- Legacy note (cleanup): if present, `packages/db/.env`, `.env.local`, `.env.production` are no longer needed when using the root `.env`. You can delete them, or update `packages/db/switch-db.ps1` to manage the root `.env` instead of the package one.

## Contribution Guidelines

1. Branching
   - Create feature branches from `main`: `feat/...`, `fix/...`, `chore/...`
2. Commits
   - Use concise, imperative messages: "feat(api): add leads endpoint"
3. Code Style
   - TypeScript, strict types; match existing formatting
   - Run `npm run lint` and `npm run check-types` before pushing
4. Database Changes
   - Edit `packages/db/prisma/schema.prisma`
   - Run `npm run db:migrate` for iterative dev migrations
   - Run `npm run db:deploy` for applying migrations
   - Provide seeds in `prisma/seed.ts` when needed
5. Testing locally
   - Ensure Docker Postgres is running and seed data is loaded
6. PRs
   - Link issue, describe scope, include screenshots for UI

## API Endpoints (summary)

- Health: `GET /health`
- Users: `GET /api/users`, `POST /api/users`
- Leads: `GET /api/leads`, `POST /api/leads`
- Contacts: `GET /api/contacts`, `POST /api/contacts`
- Campaigns: `GET /api/campaigns`, `POST /api/campaigns`
- Analytics: `GET /api/analytics/events`, `POST /api/analytics/events`

## Troubleshooting

- Postgres connection errors: verify Docker is up and port `5433` is not in use
- Prisma errors: re-run `npm run db:generate` and `npm run db:deploy`
- Type errors: run `npm run check-types` in root and per-app

## License

Private project. All rights reserved.
