# Free-Tier Hosting Guide

Everything needed to run the portfolio and its apps on free hosting, using
**Cloudflare Workers**, **Render Free** and **Neon** — with **$0 recurring cost**.

> Full architecture, per-app setup, checklists, troubleshooting and migration
> notes live in this file. Each repository also has a short `DEPLOYMENT.md`
> pointing back here.

---

## 1. Architecture

```
                        ┌──────────────────────────────────────────┐
   Browser ────────────▶│  Cloudflare Workers  (portfolio-FE)      │
   (portfolio-fe.       │  Next.js 16 via vinext                   │
    workers.dev)        │  • SSR  • /api/* reverse proxy          │
                        └───────────────┬──────────────────────────┘
                                        │  same-origin proxy
                                        │  (keeps the auth cookie
                                        │   first-party)
                                        ▼
                        ┌──────────────────────────────────────────┐
                        │  Render Free  (portfolio-BE)             │
                        │  NestJS 11 on 0.0.0.0:$PORT             │
                        │  /health  •  /health/ready               │
                        └───────────────┬──────────────────────────┘
                                        │  pooled connection
                                        ▼
                        ┌──────────────────────────────────────────┐
                        │  Neon Free  (PostgreSQL)                 │
                        │  Prisma 7 + @prisma/adapter-pg          │
                        └──────────────────────────────────────────┘
```

### Why the API is proxied through the Worker

The admin panel authenticates with an httpOnly JWT cookie that is
`SameSite=Lax`. The Worker (`*.workers.dev`) and the API (`*.onrender.com`)
are **different registrable domains**, so a direct browser call to the API
would be cross-site and the browser would drop the cookie — the admin panel
would fail to log in with no visible error.

`next.config.ts` therefore rewrites `/api/*` to the API origin. The browser
only ever talks to its own origin, the cookie stays first-party, and vinext
forwards the `Cookie` header to the upstream API (it prints this at build
time: *"Found 1 external rewrite that proxy requests to external origins"*).

**Consequence:** `NEXT_PUBLIC_API_URL` must stay **unset** in production.
The browser uses the relative `/api` path, which the rewrite proxies.

### Repository layout

Two repositories, each independently deployable:

| Repository | Local path | Stack | Target |
|---|---|---|---|
| `portfolio-FE` | `islam/client` | Next.js 16.2.12, React 19.3.1, Tailwind 4 | Cloudflare Workers |
| `portfolio-BE` | `islam/server` | NestJS 11.1.28, Prisma 7.9.1, PostgreSQL | Render Free + Neon |

Plus two static-ish Nuxt apps:

| Repository | Local path | Stack | Target |
|---|---|---|---|
| Learn Hub | `learn` | Nuxt 4.4.8 (`ssr: false`) | Cloudflare Workers (Nitro `cloudflare_module`) |
| Software Engineer Docs | `software-engineer-docs` | Nuxt 4.5.2 + Nuxt Content 3 | Cloudflare Workers (prerendered static assets) |

---

## 2. Environment variables

### Cloudflare Workers — portfolio-FE

| Variable | Type | Required | Value |
|---|---|---|---|
| `API_URL` | **Build** variable | **Yes** | `https://portfolio-be.onrender.com` |

`API_URL` is read at **build time** by `next.config.ts` to build the rewrite
destination. It must be a **build variable** in Workers Builds, not a runtime
secret. If it is missing, the build emits no rewrite and `/api/*` is not
proxied (the app still boots, but the admin panel cannot authenticate).

`NEXT_PUBLIC_API_URL` is intentionally **not** set in production — see above.

### Render Free — portfolio-BE

| Variable | Required | Notes |
|---|---|---|
| `DATABASE_URL` | **Yes** | Neon **pooled** connection string (host contains `-pooler`) |
| `JWT_SECRET` | **Yes** | `openssl rand -base64 48` |
| `FRONTEND_URL` | **Yes** | Comma-separated CORS allow-list, e.g. `https://portfolio-fe.<sub>.workers.dev` |
| `APP_URL` | Recommended | Single origin used to build magic-link emails |
| `NODE_ENV` | Yes | `production` (Render sets it for you) |
| `PORT` | Auto | Render injects it; the app listens on `0.0.0.0:$PORT` |
| `TRUST_PROXY` | Recommended | `1` — Render is behind one reverse proxy |
| `COOKIE_DOMAIN` | No | Leave empty so the cookie stays host-only |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `FROM_EMAIL` | Only for magic links | |
| `SEED_ADMIN_PASSWORD` | Only for seeding | Read by `prisma/seed.ts` |
| `DIRECT_DATABASE_URL` | Only for migrations | Neon **direct** URL; see §5 |

### Neon

Neon has no environment variables of its own — it issues connection strings.
You need **two**:

- **Pooled** (`ep-xxx-pooler.<region>.aws.neon.tech`) → runtime traffic.
  Goes into `DATABASE_URL`.
- **Direct** (`ep-xxx.<region>.aws.neon.tech`) → migrations. Prisma Migrate
  cannot run over a transaction-mode pooler.

Both end with `?sslmode=require`.

### Security rules

- Real secrets live **only** in Render / Cloudflare dashboards and in local
  `.env` / `.env.local` files, which are git-ignored.
- `.env.example` files are committed and contain placeholders only.
- Never put `DATABASE_URL`, `JWT_SECRET` or SMTP credentials in a
  `NEXT_PUBLIC_*` variable — those are inlined into the browser bundle.

---

## 3. Manual dashboard setup

### 3.1 Neon (do this first — the API needs the database to boot)

1. Sign in at <https://neon.tech> and create a project named `portfolio`.
2. Pick the **Free** plan. Choose the region closest to your Render region
   (Oregon / `aws-us-east-1` pairs with Render's `oregon`).
3. Neon creates a `neondb` database with a `neondb_owner` role.
4. Open **Connection details** and copy **both** strings:
   - *Pooled connection* → `DATABASE_URL`
   - *Direct connection* → `DIRECT_DATABASE_URL`
5. **Do not delete the project.** Free projects are kept for a while, but the
   compute scales to zero after ~5 minutes of inactivity.

### 3.2 Render (the API)

1. Sign in at <https://render.com> and connect the `portfolio-BE` repository.
2. **New → Web Service**, then:
   - **Name:** `portfolio-be`
   - **Region:** Oregon
   - **Branch:** `main`
   - **Root Directory:** *(leave empty)*
   - **Runtime:** Node
   - **Build Command:** `npm ci && npm run build`
   - **Start Command:** `npm run start:prod`
   - **Instance Type:** Free
3. **Health Check Path:** `/health`
4. Add the environment variables from §2 under **Environment**.
   `JWT_SECRET` can be generated by Render.
5. **Create Web Service**.

> `render.yaml` in the repo is a ready-made Blueprint. Render's
> **New → Blueprint** flow reads it and pre-fills everything; you only have to
> supply the `sync: false` values. Using the Blueprint is optional — the
> manual route above works identically.

`npm ci` installs dependencies, then the `prebuild` script runs
`prisma generate` automatically, then `nest build` compiles to `dist/`.

### 3.3 Cloudflare (the portfolio)

1. Sign in at <https://dash.cloudflare.com>.
2. **Workers & Pages → Create → Import a project** (Workers Builds) and
   connect the `portfolio-FE` repository.
3. Build settings:
   - **Framework preset:** None (or "None" for Workers)
   - **Build command:** `npm run build:vinext`
   - **Build output directory:** *(leave empty — vinext writes to
     `.cloudflare/output`)*
   - **Root directory:** *(leave empty)*
   - **Node version:** `22`
4. **Settings → Variables and Secrets → Add**:
   - `API_URL` → **Type: Text** and **under "Build variables"** — this is the
     important part. The rewrite destination is baked in at build time.
   - `NODE_ENV` → `production` (Text)
5. **Save and deploy.**

Your portfolio will be at `https://portfolio-fe.<your-subdomain>.workers.dev`.

### 3.4 Cloudflare (Learn Hub)

Same as above, but for the `learn` repository:

- **Build command:** `npm run build:cloudflare`
- No environment variables needed.

### 3.5 Cloudflare (Software Engineer Docs)

Same, but for `software-engineer-docs`:

- **Build command:** `npm run build:cloudflare`
- No environment variables needed.

---

## 4. GitHub CI/CD

### Continuous deployment (already configured)

| App | Mechanism | Trigger |
|---|---|---|
| portfolio-FE | **Cloudflare Workers Builds** (GitHub integration) | push to `main` |
| portfolio-BE | **Render auto-deploy** (GitHub integration) | push to `main` |
| Learn Hub | **Cloudflare Workers Builds** | push to `main` |
| Engineer Docs | **Cloudflare Workers Builds** | push to `main` |

Enable them:

- Cloudflare: in the Worker → **Settings → Builds → Connect repository**.
- Render: in the Web Service → **Settings → Auto-Deploy → Yes**.

### Continuous integration (added in this change)

`.github/workflows/ci.yml` was added to `portfolio-FE` and `portfolio-BE`:

- **portfolio-FE** — lint → `tsc --noEmit` → `next build` →
  `vinext check` → `vite build` (the Cloudflare build).
- **portfolio-BE** — lint → `build` → unit tests → e2e tests, against a
  throwaway PostgreSQL 16 service container.

These run on every push and pull request to `main`.

### Branch strategy

`main` is the production branch. The current working branches
(`feature/button-loaders`, `fix/experience-date-normalization`) are feature
branches — merge them into `main` before you want them live.

---

## 5. Database migrations

Migrations are **never** run automatically against production. A migration
that runs on every deploy can lock a table, fail halfway, or drop data.

### One-time setup

```bash
cd islam/server

# Direct (non-pooled) Neon URL, because Prisma Migrate cannot use the pooler
export DIRECT_DATABASE_URL="postgresql://USER:PASSWORD@ep-xxx.REGION.aws.neon.tech/neondb?sslmode=require"

npm run db:deploy    # applies prisma/migrations/* to the database
npm run db:seed      # optional: admin user + sample content
```

`prisma.config.ts` uses `DIRECT_DATABASE_URL` when present and falls back to
`DATABASE_URL`, so the same command works for both.

### Day-to-day

```bash
# 1. Create a migration locally (against a dev database)
npm run db:migrate            # prompts for a migration name

# 2. Review the generated SQL in prisma/migrations/<timestamp>_*/
#    This is the file that will run in production. Read it.

# 3. Apply it to production
DIRECT_DATABASE_URL="postgresql://...direct..." npm run db:deploy
```

Useful commands:

| Command | Purpose |
|---|---|
| `npm run db:generate` | Regenerate the Prisma client (runs automatically before `build`) |
| `npm run db:status` | Show which migrations are applied |
| `npm run db:studio` | Open Prisma Studio against `DATABASE_URL` |
| `npm run db:seed` | Seed the admin user and sample content (idempotent) |

---

## 6. Deployment checklist

### First deployment

- [ ] Create the Neon project; copy the **pooled** and **direct** URLs.
- [ ] Apply migrations locally against Neon: `DIRECT_DATABASE_URL=... npm run db:deploy`
- [ ] Seed the database: `npm run db:seed` (then change the seeded password)
- [ ] Create the Render Web Service from `portfolio-BE`; fill in the env vars
- [ ] Verify: `curl https://portfolio-be.onrender.com/health` → `{"status":"ok",...}`
- [ ] Verify: `curl https://portfolio-be.onrender.com/health/ready` → `"database":"reachable"`
- [ ] Create the Cloudflare Worker from `portfolio-FE`
- [ ] Add `API_URL=https://portfolio-be.onrender.com` as a **build variable**
- [ ] Deploy and open `https://portfolio-fe.<sub>.workers.dev`
- [ ] Open `/admin/login` and sign in — **this is the real end-to-end test**
      of the cookie proxy. If the cookie is not forwarded, the API's
      `FRONTEND_URL` is wrong or `API_URL` was set as a runtime secret
      instead of a build variable.
- [ ] Fill the live URLs into `data/projects.ts` → `PROJECT_LIVE_URLS`
- [ ] Add a screenshot per project to `public/projects/<slug>.png`
- [ ] Deploy Learn Hub and Engineer Docs to Cloudflare
- [ ] Add their URLs to `PROJECT_LIVE_URLS`
- [ ] Push the `main` branches so CI runs

### Every subsequent deploy

- [ ] `npm run build` + `npm test` locally (both repos)
- [ ] If the schema changed: create a migration, review the SQL, then
      `npm run db:deploy` against production **before** deploying the code
- [ ] Push to `main`
- [ ] Watch the Cloudflare / Render deploy logs

---

## 7. Troubleshooting

### The admin panel logs in but immediately logs out

The cookie is not surviving the trip to the browser.

1. Check the API's `FRONTEND_URL` contains the **exact** Worker origin
   (scheme + host, no trailing slash).
2. Check `API_URL` on Cloudflare is a **build variable**, not only a runtime
   secret. The rewrite destination is baked in at build time — rebuild after
   changing it.
3. Confirm in the browser devtools → Network → the `/api/auth/login`
   response has a `Set-Cookie` header. If it is missing, the request never
   reached the API.
4. Confirm the request to `/api/auth/login` was made **same-origin**
   (the Request URL should start with your Worker origin, not
   `onrender.com`). If it points at `onrender.com`, `NEXT_PUBLIC_API_URL`
   is set — remove it.

### `npm run build:vinext` succeeds but `/api/*` returns 404

`API_URL` was not present at build time. `next.config.ts` deliberately
registers **no** rewrite in production when `API_URL` is missing, so the
failure is loud rather than silently proxying to `localhost`. Set the build
variable and redeploy.

The build log tells you what happened:

```
[vinext] Found 1 external rewrite that proxy requests to external origins:
  /api/:path* → https://portfolio-be.onrender.com/:path*
```

If that line is missing, `API_URL` was not picked up.

### Render deploy fails with "Cannot find module ... .prisma/client"

`prisma generate` did not run. It is wired as a `prebuild` script, so it
runs automatically on `npm run build`. If you overrode the build command to
just `nest build`, restore `npm ci && npm run build`.

### Render service crashes on boot with "Missing required environment variable(s)"

`src/config/env.validation.ts` fails the boot in production when
`DATABASE_URL`, `JWT_SECRET` or `FRONTEND_URL` is missing. This is
intentional — a half-configured service fails loudly instead of serving
broken responses. Add the variable and redeploy.

### The API returns 500 on every database route, but `/health` is fine

`/health` deliberately does not touch the database, so a green `/health`
with failing data routes means the database is unreachable. Check:

- `DATABASE_URL` points at the **pooled** Neon URL.
- The Neon project has not been deleted or paused.
- `curl .../health/ready` returns `"database":"unreachable"` to confirm.

### First request after 15 minutes returns a 502 / times out

This is the Render free tier sleeping. The service takes ~50 seconds to wake.
Hit the URL again, or use a cron ping (see §9).

### Neon's pooled URL times out under load

The pooler is in transaction mode. Do not use interactive transactions or
`LISTEN/NOTIFY` over it. `DATABASE_POOL_MAX` defaults to `10`; lower it if
Neon reports connection limits.

### `vinext check` reports a new incompatibility

Cloudflare's [vinext compatibility dashboard](https://vinext.dev) tracks
feature support. Run `npm run check:vinext` after upgrading Next.js.

### The build works locally but fails on Cloudflare

- Cloudflare builds on Linux; a Windows-only native module will fail.
- `npm ci` is used on Cloudflare, so `package-lock.json` must be committed
  and in sync with `package.json`.
- Set `NODE_VERSION` to `22` in the Cloudflare build settings.

### Want to fall back from vinext to OpenNext

`@opennextjs/cloudflare@1.20.12` declares
`next: '>=15.5.28 <16 || >=16.4.1'`. The project is on `next@16.2.12`,
which is **outside** that range, so OpenNext is not currently an option
without upgrading Next.js to `>=16.4.1`. If vinext becomes a blocker:

1. `npm i next@16.4.2 eslint-config-next@16.4.2`
2. `npm i -D @opennextjs/cloudflare wrangler`
3. Add `open-next.config.ts` and a `wrangler.jsonc` with
   `main: ".open-next/worker.js"`.
4. Re-run `npm run check:vinext` to confirm the upgrade did not break
   anything else.

Keep the original `next dev` / `next build` scripts — they are the
fastest local feedback loop and are independent of the deploy adapter.

---

## 8. Free-tier limits and what they mean

### Cloudflare Workers (Free)

| Limit | Value | Impact here |
|---|---|---|
| Requests | 100,000 / day | Ample for a portfolio |
| CPU time | 10 ms (free tier, per request) | SSR pages are cheap; heavy API calls are proxied, not computed |
| Worker size | 3 MB (compressed) | The portfolio bundle is well under this |
| Static assets | 20,000 files | Not a constraint |
| Subdomains | `*.workers.dev` | Free URL, no custom domain needed |
| Deployments | Unlimited | — |

### Render Free

| Limit | Value | Impact here |
|---|---|---|
| Sleep | After 15 min idle | ~50 s wake on the next request |
| Monthly hours | 750 (≈ 31 days of 1 instance) | Effectively unlimited for one instance |
| Instance | 0.1 CPU, 512 MB RAM | Comfortable for a small NestJS API |
| Static sites | Free, unlimited | Could host the portfolio instead — but SSR is needed here |
| Cold start | ~50 s | Use a ping to keep it warm (below) |

### Neon Free

| Limit | Value | Impact here |
|---|---|---|
| Storage | 0.5 GB per project | Ample for a portfolio's content |
| Compute | Scales to zero after ~5 min idle | First query after idle pays a cold-start penalty |
| Projects | 1 (free) | Use one project with multiple branches if you need isolation |
| Connections | Pooled endpoint handles bursts | Use the pooled URL, never the direct one, at runtime |

### Keeping the API awake (optional)

Render's free tier sleeping is the most visible limitation. A free
cron service (e.g. cron-job.org, or GitHub Actions on a schedule) that hits
`https://portfolio-be.onrender.com/health` every 10 minutes keeps it warm:

```yaml
# .github/workflows/keep-warm.yml
name: Keep Render awake
on:
  schedule:
    - cron: "*/10 * * * *"
  workflow_dispatch:
jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - run: curl -fsS https://portfolio-be.onrender.com/health
```

Note: GitHub Actions schedules can be delayed by up to ~15 minutes on busy
public runners, so this may not fully defeat the sleep.

### Cost summary

| Service | Plan | Cost |
|---|---|---|
| Cloudflare Workers | Free | $0 |
| Render | Free | $0 |
| Neon | Free | $0 |
| GitHub Actions | Free (2,000 min/mo for private, unlimited for public) | $0 |
| **Total** | | **$0 / month** |

---

## 9. Adding a new project to the portfolio

There are two ways. Use the admin panel for content you want to edit later
without a deploy; use the static catalogue for code-controlled content.

### Option A — the admin panel (no deploy needed)

1. Sign in at `https://portfolio-fe.<sub>.workers.dev/admin/projects`.
2. Fill in title, description, tech stack (comma-separated), screenshot URL,
   live demo URL and GitHub URL.
3. Toggle **Featured** to show it in the home-page strip.
4. Save. The change is live immediately (the pages are SSR and the API is
   read on every request).

### Option B — the static catalogue (version controlled)

Edit `islam/client/data/projects.ts`:

```ts
export const staticProjects: PortfolioProject[] = [
  {
    slug: "my-new-app",
    title: "My New App",
    description: "What it does, in one or two sentences.",
    techs: ["Next.js", "NestJS", "PostgreSQL"],
    imageUrl: "/projects/my-new-app.png",   // file in public/projects/
    liveUrl: PROJECT_LIVE_URLS.myNewApp || undefined,
    githubUrl: "https://github.com/islamAbuSleem/my-new-app",
    featured: false,
    order: 8,
  },
];
```

Add the live URL in one place:

```ts
export const PROJECT_LIVE_URLS = {
  ...
  myNewApp: "https://my-new-app.onrender.com",
};
```

Then add a screenshot at `islam/client/public/projects/my-new-app.png` and
push. The image can be any size; the card crops it to 16:9.

### Which wins?

`lib/projects.ts` merges the two sources by title. A database project with
the same title **overrides** the static entry, so the admin panel always has
the final say. Static entries that are not in the database are appended
after the database ones.

### Deploying a new app to a free host

| App type | Host | Command |
|---|---|---|
| Next.js | Cloudflare Workers | `npx vinext init --platform=cloudflare` then `npm run build:vinext` + `npm run deploy:vinext` |
| Nuxt | Cloudflare Workers | `npm run build:cloudflare` (Nitro `cloudflare_module`) + `npm run deploy:cloudflare` |
| NestJS | Render | Create a Web Service; build `npm ci && npm run build`, start `npm run start:prod`, health `/health` |
| Any Node server | Render | Same as NestJS; add a `Procfile` with `web: npm start` |
| Spring Boot | Render | Build `./mvnw package -DskipTests`, start `java -jar target/*.jar` |

For a Spring Boot or other JVM app, set `PORT` and bind to `0.0.0.0` — the
same rule as the NestJS service.

---

## 10. Future migration to a VPS (Coolify)

The setup was chosen so that moving off the free tiers is a re-point, not a
rewrite.

- **Frontend** — `npm run build` produces a standard Next.js build; on a VPS
  run `next start` behind a reverse proxy, or keep the Worker.
- **Backend** — `npm run start:prod` runs `node dist/main` on any Node
  host. It already binds `0.0.0.0:$PORT` and reads `PORT` from the
  environment, which is exactly what Coolify, Docker and Kubernetes
  expect. `Procfile` is committed for Heroku-style platforms.
- **Database** — `DATABASE_URL` is a standard PostgreSQL URL. Point it at
  Coolify's managed Postgres (or a self-hosted container) and nothing else
  changes. The Prisma schema and migrations are portable as-is.
- **CORS** — `FRONTEND_URL` is a variable, so adding the new domain is a
  one-line change.
- **Workers-specific code** — only `vite.config.ts` and
  `cloudflare.config.ts` are Cloudflare-bound. Both can be deleted without
  touching the application.

WordPress can later be added as a third service (Render or the same VPS)
behind its own hostname; the portfolio's CORS allow-list and `FRONTEND_URL`
are already variable-driven, so adding an origin is a configuration change.
