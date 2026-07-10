# weekly-report-app
Full-stack weekly report generator and team dashboard with role-based access, built with Next.js, Express, Prisma, and PostgreSQL.

# Logbook — Weekly Report Generator & Team Dashboard

A full-stack app for submitting structured weekly work reports and analyzing them
across a team. Team members log a fixed-structure weekly report; managers get a
dashboard with filters, charts, and an AI assistant for querying team activity.

- **Frontend:** Next.js (App Router) + Tailwind CSS + Recharts
- **Backend:** Node.js + Express + Prisma ORM
- **Database:** PostgreSQL
- **Auth:** JWT + bcrypt, role-based access control (`MEMBER` / `MANAGER`)
- **AI Assistant (optional):** Google Gemini API over stored report context

See `docs/architecture.md` for the system design write-up and `docs/er-diagram.svg`
/ `docs/er-diagram.mmd` for the database schema diagram.

---

## 1. Prerequisites

- **Node.js** 18+
- **Docker Desktop** (easiest way to run PostgreSQL locally — see step 3), OR
  a PostgreSQL 14+ install of your own
- npm (comes with Node.js)

---

## 2. Install dependencies

```bash
# Backend
cd backend
npm install

# Frontend (in a separate terminal)
cd frontend
npm install
```

---

## 3. Running the database (PostgreSQL)

The simplest way is a Docker container — no local Postgres install needed.

```bash
docker run --name weekly-reports-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=weekly_reports -p 5432:5432 -d postgres:16
```

This downloads and starts Postgres in the background. Verify it's running:

```bash
docker ps
```

You should see `weekly-reports-db` with status `Up`.

**Next time** you just need to start the existing container (no need to re-run
the command above):

```bash
docker start weekly-reports-db
```

*(Alternative: install PostgreSQL directly from [postgresql.org](https://www.postgresql.org/download/) instead of using Docker, and create a database named `weekly_reports`.)*

---

## 4. Configure the backend

```bash
cd backend
cp .env.example .env
```

Edit `.env`:

```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/weekly_reports?schema=public"
JWT_SECRET="pick-any-long-random-string"
PORT=4000
CORS_ORIGIN="http://localhost:3000"

# Optional — see section 7 below
GEMINI_API_KEY=
```

---

## 5. Create the schema and seed demo data

```bash
cd backend
npx prisma migrate dev --name init
npm run prisma:seed
```

The seed script creates:
- Manager: `manager@demo.com` / `password123`
- Members: `ava@demo.com`, `leo@demo.com`, `priya@demo.com` / `password123`
- 3 sample projects and ~3 weeks of sample reports

---

## 6. Run the app

**Backend** (Terminal 1):
```bash
cd backend
npm run dev
```
Listens on `http://localhost:4000`. Health check: `GET http://localhost:4000/api/health` → `{"status":"ok"}`.

**Frontend** (Terminal 2 — new window, keep the backend running):
```bash
cd frontend
cp .env.local.example .env.local
npm run dev
```
Visit `http://localhost:3000`. You'll land on `/login`.

---

## 7. (Optional) Enable the AI chat assistant

The "Ask about the team" panel on the manager dashboard calls the **Google
Gemini API**, chosen because it has a free tier with no billing setup required.

1. Get a free key at [aistudio.google.com/api-keys](https://aistudio.google.com/api-keys) (Google account, no card needed)
2. Add it to `backend/.env`:
   ```
   GEMINI_API_KEY=AIza...
   ```
3. Restart the backend (`Ctrl+C`, then `npm run dev` again)

Without a key set, the endpoint returns a friendly "not configured" message
instead of failing — the rest of the app works normally either way.

---

## Project structure

```
backend/
  prisma/schema.prisma       # DB schema (User, Project, ProjectAssignment, Report)
  prisma/seed.js              # Demo data
  src/
    routes/                   # Express route definitions
    controllers/               # Request handlers / business logic
    middleware/auth.js          # JWT auth + role-based authorize()
    utils/week.js                # Monday–Sunday week normalization
    app.js / server.js

frontend/
  app/
    login/ register/           # Auth pages
    reports/                    # Personal weekly report page (Team Member)
    dashboard/                   # Team dashboard with charts + AI assistant (Manager)
    projects/                     # Project/category management (Manager)
  components/                    # Reusable UI: forms, cards, charts, nav, auth guard
  lib/                            # API client, auth context, week helpers
```

---

## Notes on design decisions

- **Fixed report structure:** the report form's fields and order are hardcoded
  in `ReportForm.js` / the `Report` Prisma model, not user-configurable, so every
  team member's report stays directly comparable on the manager dashboard.
- **Weeks are normalized** server- and client-side to the Monday of the
  calendar week (`utils/week.js` / `lib/week.js`), so reports always line up
  across the team regardless of which day someone files on.
- **Role-based access control** is enforced on the backend (`authorize('MANAGER')`
  middleware on every manager-only route) rather than only hidden in the UI.

---

## Troubleshooting

- **`EADDRINUSE: address already in use :::4000`** — another process is already
  using port 4000. Find and stop it:
  ```bash
  netstat -ano | findstr :4000      # Windows — note the PID in the last column
  taskkill /PID <that_pid> /F        # Windows
  # or on macOS/Linux:
  lsof -i :4000
  kill -9 <that_pid>
  ```
  Then retry `npm run dev`.
- **`npm install` fails with `ENOTFOUND` / network errors** — usually a local
  network/DNS/VPN issue, not a project problem. Try `ping registry.npmjs.org`;
  if that fails, check your Wi-Fi/VPN/DNS settings and retry.
- **AI assistant says "not configured"** — `GEMINI_API_KEY` is missing or the
  backend wasn't restarted after adding it. Stop the backend fully, confirm the
  key is saved in `.env`, then start it fresh.
