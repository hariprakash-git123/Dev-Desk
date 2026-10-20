# DevDesk

A developer command center: GitHub activity, your own projects and tasks, and website health
checks in one dashboard, with an optional AI briefing.

## Status

**Phases 1 to 6 of 7 are built.**

| Works now                                                    | Not built yet                            |
| ------------------------------------------------------------ | ---------------------------------------- |
| Sign-in with GitHub, sessions, sign-out, account deletion    | A live deployment (Phase 7)              |
| Choosing repositories; live issues and pull requests         | Review and check status on pull requests |
| Projects and tasks, saved to the database, with linked repos | Overdue-task and browser notifications   |
| Activity feed: GitHub events, projects, tasks and incidents  | End-to-end browser tests (Phase 7)       |
| Website monitoring: scheduled checks, history, incidents     |                                          |
| Briefings built from your own facts, with source links       |                                          |
| In-app incident notifications                                |                                          |
| Tests proving two users cannot reach each other's data       |                                          |

Things to know:

- **Briefings are off by default.** `AI_PROVIDER=preview` builds them locally with DevDesk's own
  templates, labelled "Preview — not AI-written", with no AI and no cost. Real AI briefings need
  `AI_PROVIDER=anthropic` and an API key. See [docs/ai-briefings.md](docs/ai-briefings.md),
  including exactly what would be sent to Anthropic.
- **Monitoring needs the worker running.** `npm run dev` starts it. See
  [docs/monitoring.md](docs/monitoring.md).
- **Deployment files are prepared but not yet used.** [docs/deploy-oracle.md](docs/deploy-oracle.md)
  sets DevDesk up on one Oracle Cloud Always Free VM, with encrypted backups. It has not been run
  on a real VM, and commercial use of Always Free is not confirmed.
- **Sign-in needs a GitHub App that you create.** Follow
  [docs/github-app-setup.md](docs/github-app-setup.md).

The full plan is in [docs/architecture.md](docs/architecture.md).

## Run it locally

You need [Node.js 22](https://nodejs.org) or newer and
[Docker Desktop](https://www.docker.com/products/docker-desktop/) (for the database).

```bash
npm install
```

Create your local settings file, then put a fresh encryption key in it:

```bash
cp .env.example .env
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

Paste the printed value after `TOKEN_ENCRYPTION_KEY=` in `.env`. Then start the database, create
its tables, and run the app:

```bash
npm run db:up
npm run db:deploy
npm run dev
```

Open http://localhost:5173. The API runs on port 4000, and the page reaches it through `/api`.

## Commands

| Command                | What it does                                              |
| ---------------------- | --------------------------------------------------------- |
| `npm run dev:worker`   | Start only the monitoring worker                          |
| `npm run dev`          | Start the API, the frontend and the monitoring worker     |
| `npm run dev:web`      | Start only the frontend                                   |
| `npm run dev:api`      | Start only the API                                        |
| `npm run db:up`        | Start the local PostgreSQL container                      |
| `npm run db:deploy`    | Apply existing migrations to the database                 |
| `npm run db:status`    | Show whether every migration is applied                   |
| `npm run db:migrate`   | Create a new migration after editing the schema           |
| `npm run ai:smoke`     | One real, billed AI request; needs `AI_SMOKE=1` and a key |
| `npm run test`         | Run all tests (the API tests need the database running)   |
| `npm run typecheck`    | Check TypeScript types                                    |
| `npm run lint`         | Run ESLint                                                |
| `npm run format`       | Format every file with Prettier                           |
| `npm run format:check` | Check formatting without changing files                   |
| `npm run build`        | Build the frontend for production                         |

The API tests use a separate database, `devdesk_test`, which they create and erase themselves.
They refuse to run against any database whose name does not end in `_test`.

## Layout

```
apps/web/            React + Vite + Tailwind frontend
apps/api/            Express API: sign-in, sessions, projects, tasks
apps/worker/         Runs scheduled website checks
packages/database/   Prisma schema, migrations and database client
packages/monitoring/ Safe URL checks, incident rules and the check scheduler
packages/shared/     Types and Zod validation shared by the frontend and the API
deploy/              Caddy, systemd, settings templates, backup and restore scripts
docs/                Architecture plan, GitHub App setup and deployment guide
.github/workflows/   Continuous integration
```

## How your data is protected

- The browser never holds a GitHub token. It holds one `HttpOnly` session cookie.
- GitHub tokens are encrypted (AES-256-GCM) before they are stored. The key lives in `.env`, not
  in the database.
- A repository is tracked only after GitHub confirms that your own token can read it. The page
  sends an id; every other detail comes from GitHub.
- Every query for user-owned data filters on the signed-in user's id. Asking for someone else's
  record gets the same "not found" answer as asking for one that does not exist.
- Requests that change data must come from the DevDesk page itself (origin check plus a custom
  header), which blocks cross-site request forgery.

## Where data comes from

| Module in `apps/web/src/lib` | Used by                                | Source                        |
| ---------------------------- | -------------------------------------- | ----------------------------- |
| `workspace-api.ts`           | Today, Projects, Tasks, Activity       | DevDesk API and database      |
| `github-api.ts`              | Repositories, Issues and pull requests | DevDesk API, read from GitHub |
| `monitoring-api.ts`          | Monitoring, Today, project pages       | DevDesk API and database      |
| `briefing-api.ts`            | Briefing                               | DevDesk API and database      |
| `notifications-api.ts`       | Notification bell                      | DevDesk API and database      |

No page shows sample data any more.

The activity feed records three things you do in DevDesk (creating a project, creating a task and
completing a task) and when a monitored site's incident opens or resolves. Other edits are not recorded. GitHub events arrive when a repository is read,
which happens when you open it or choose **Refresh from GitHub**.
