# DevLens

Look up any GitHub developer, explore their profile and repositories, and compare several
developers head-to-head on stars, languages and commit activity. Signed-in users can pin
repositories with private notes and tags, and save named comparisons.

**Stack:** React 19 + TypeScript + Vite · Node.js + Express 5 + TypeScript · PostgreSQL 16

> Design decisions, the GitHub rate-limit strategy and trade-offs are in
> **[docs/TECHNICAL_NOTES.md](docs/TECHNICAL_NOTES.md)**.

## Features

- **Auth:** email + password sign up / log in. bcrypt hashes, session JWT in an `httpOnly` cookie. Every user's pins and comparisons are private to them.
- **Developer profile:** avatar, bio and followers; total stars, forks, average stars per repo; language breakdown (by bytes of code or by repo count); 52-week commit activity chart.
- **Repositories:** filterable, sortable, paginated list (stars, primary language, last updated), plus a detail page with full language breakdown and weekly commits.
- **Compare:** 2–5 developers side by side, with a metrics table that marks who leads each metric, bar charts, a monthly commits chart and each developer's language mix. A bad username is reported inline without breaking the rest of the comparison.
- **Curate:** pin repositories with notes and tags, filter pins by tag or text, and save, rename, edit or delete named comparisons (full CRUD).
- **Responsive:** phone layout with a bottom tab bar; the compare table scrolls behind a sticky metric column when there are more developers than fit.
- **Fast and within GitHub's limits:** Postgres cache with stale-while-revalidate, ETag conditional requests, coalescing of identical in-flight lookups, a concurrency limiter and a rate-limit budget. See the technical notes.

## Quick start

Prerequisites: **Node.js ≥ 20**, **Docker** (for PostgreSQL), or an existing PostgreSQL ≥ 13.

```bash
# 1. Install dependencies (npm workspaces: server + client)
npm install

# 2. Start PostgreSQL (docker-compose, port 5432)
npm run db:up

# 3. Configure the API
cp server/.env.example server/.env
#    then edit server/.env and set GITHUB_TOKEN (strongly recommended, see below)

# 4. Run API (http://localhost:4000) and web app (http://localhost:5173) together
npm run dev
```

Open **http://localhost:5173**, create an account and search for a username such as `torvalds`.
Database migrations run automatically when the API starts (`npm run migrate` runs them on their own).

### GitHub token

Without a token GitHub allows **60 requests/hour**; with one it allows **5,000**. Create a
[personal access token](https://github.com/settings/tokens). Classic with no scopes, or
fine-grained with "Public repositories (read-only)", is enough. Put it in `server/.env` as
`GITHUB_TOKEN`. The token stays on the server and is never sent to the browser.

### Production (single container)

```bash
GITHUB_TOKEN=ghp_xxx JWT_SECRET=$(openssl rand -hex 32) docker compose --profile app up --build
# -> http://localhost:4000 (API and built SPA served from one origin)
```

Or without Docker: `npm run build && NODE_ENV=production npm start`.

## Environment variables (`server/.env`)

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | yes | – | PostgreSQL connection string |
| `JWT_SECRET` | yes | – | Secret for signing session tokens (≥ 32 chars) |
| `GITHUB_TOKEN` | recommended | – | GitHub token; raises the limit from 60 to 5,000 requests/hour |
| `PORT` | no | `4000` | API port |
| `NODE_ENV` | no | `development` | `production` enables secure cookies and serves `client/dist` |
| `CORS_ORIGIN` | no | `http://localhost:5173` | Only needed if the SPA is served from a different origin |
| `PROFILE_TTL_MINUTES` | no | `30` | How long a cached profile and repo list count as fresh |
| `ACTIVITY_TTL_MINUTES` | no | `360` | How long cached language and commit data count as fresh |
| `ACTIVITY_REPO_LIMIT` | no | `10` | Repos per developer analysed for languages and commits |
| `MAX_REPO_PAGES` | no | `5` | Max pages of 100 repos imported per developer |
| `GITHUB_CONCURRENCY` | no | `8` | Max simultaneous GitHub requests per server instance |
| `GITHUB_RATE_LIMIT_RESERVE` | no | `100` | Requests held back for user lookups; background refreshes stop below this |

The client needs no environment variables. In development Vite proxies `/api` to
`http://localhost:4000` (set `API_URL` to change this).

## Scripts

| Command | What it does |
|---|---|
| `npm run dev` | API (tsx watch) + client (Vite) with hot reload |
| `npm test` | Server unit tests (Vitest) |
| `npm run typecheck` | Type-check server and client |
| `npm run build` | Compile server to `server/dist`, bundle client to `client/dist` |
| `npm start` | Run the compiled server |
| `npm run db:up` / `db:down` | Start / stop the PostgreSQL container |
| `npm run migrate` | Apply database migrations |

## REST API

All routes are under `/api` and return JSON. Errors have the shape `{ "error": { "code", "message", "details?" } }`.
Every route except `/auth/*` and `/health` needs a session (cookie, or `Authorization: Bearer <token>`).

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/register` | `{email, password}` → creates the account and starts a session |
| `POST` | `/auth/login` | `{email, password}` → starts a session |
| `POST` | `/auth/logout` | Clears the session |
| `GET` | `/auth/me` | Current user |
| `GET` | `/developers/:login` | Profile + aggregate metrics + freshness info |
| `POST` | `/developers/:login/refresh` | Force a refresh from GitHub (rate limited) |
| `GET` | `/developers/:login/repos` | `?sort=stars\|pushed\|forks\|name&language=&q=&forks=true&limit=&offset=` |
| `GET` | `/developers/suggest?q=` | Autocomplete from already-cached developers (no GitHub calls) |
| `GET` | `/compare?users=a,b,c` | 2–5 developers' metrics, plus who leads each metric and per-login errors |
| `GET` | `/repos/:id` | Repo detail, language breakdown, weekly commits and your pin |
| `GET` | `/pins` | Your pins; `?tag=` and `?q=` filters |
| `GET` | `/pins/tags` | Your tags with counts |
| `POST` | `/pins` | `{repoId, note?, tags?}` |
| `PATCH` | `/pins/:id` | `{note?, tags?}` |
| `DELETE` | `/pins/:id` | Unpin |
| `GET` | `/comparisons` | Your saved comparisons |
| `GET` | `/comparisons/:id` | One comparison |
| `POST` | `/comparisons` | `{name, logins[]}` |
| `PATCH` | `/comparisons/:id` | `{name?, logins?}` |
| `DELETE` | `/comparisons/:id` | Delete |
| `GET` | `/health` | DB ping + GitHub rate-limit status and request stats |

## Project structure

```
server/
  src/
    config.ts              env validation (zod)
    db/                    pg pool, migration runner, SQL migrations
    github/client.ts       GitHub HTTP client: ETags, retries, rate-limit budget, concurrency
    github/api.ts          endpoint wrappers + Postgres-backed ETag store
    services/developers.ts cache-first lookup, stale-while-revalidate, ingestion, metrics
    services/*.ts          auth, repos, pins, comparisons
    routes/                Express routers (validation with zod)
  test/                    Vitest unit tests
client/
  src/
    lib/                   API client, React Query hooks, types, formatting
    components/            layout, search, charts, repo list, pin editor
    pages/                 auth, home, developer, repo, compare, pins, comparisons
docs/TECHNICAL_NOTES.md    design decisions and trade-offs
```
