# Using Relproc Auth locally

This is for **product apps** (QMS, EDMS, Projects, Proposals, HRMS, …) that need login and permissions. Auth is a **separate service**. Do not copy this repo into your app, do not collect passwords, and do not point a local UI at production Auth.

Public API and npm install live in the root [README](../README.md). This file is the local-dev path.

## What you run

| Process | Repo | Default URL |
| --- | --- | --- |
| Auth API | `relproc-auth` | http://localhost:8787 |
| Auth login UI | `relproc-auth` | http://localhost:5174/login |
| Your API | your repo | pick a free port (HRMS `8788`, Proposals `8789`, …) |
| Your UI | your repo | pick a free port (HRMS `5173`, Proposals `5177`, …) |

Keep `relproc-auth` cloned once (for example `platform-repos/relproc-auth`) and **leave it running** next to your app. Your product repo only depends on the published `@relproc/auth` package (backend) and HTTP (frontend).

Production Auth (`https://auth.relproc.com`) will **not** work from `localhost`. The session cookie is `Domain=.relproc.com` and `Secure`. Local UI ↔ local Auth only.

Use **`localhost` everywhere**. Mixing `localhost` and `127.0.0.1` drops the cookie.

## 1. Start Auth

Ask **vinoth@relproc.com** for:

- a GitHub PAT (`read:packages` + `repo`) if you need to clone or install `@relproc/auth`
- local `JWT_SECRET` and `INTERNAL_TOKEN` (must match the Auth instance you talk to)

```bash
cd platform-repos/relproc-auth
npm install
npm run dev:api    # http://localhost:8787
npm run dev:web    # http://localhost:5174/login
```

You do not migrate or seed Auth unless you are standing up a **new** Auth database. Day-to-day product work uses the existing Auth Neon + seed.

Confirm: `GET http://localhost:8787/health` → `{ "ok": true, "service": "relproc-auth" }`.

## 2. Allow your UI origin in Auth

After login, Auth only sends the browser back to origins on `VITE_RETURN_ORIGINS`. Your API only accepts browser calls from origins on the CORS allowlist.

If your UI is `http://localhost:5178`:

**Auth web** `apps/web/.env.development` — add the origin to `VITE_RETURN_ORIGINS` (comma-separated, no trailing slash):

```
VITE_AUTH_API_URL=http://localhost:8787
VITE_DEFAULT_APP_URL=http://localhost:5173
VITE_RETURN_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5178
```

Restart `npm run dev:web` after changing this file. Vite does not pick it up otherwise.

**Auth API** — add the same origin to CORS:

- `apps/api/src/utils/cors.ts` → `LOCAL_DEV_ORIGINS`, and/or
- `apps/api/wrangler.jsonc` → `vars.CORS_ORIGINS`

Local Auth currently treats `wrangler.jsonc` `ENVIRONMENT` as `production` unless `.dev.vars` overrides it, so **put the origin in `CORS_ORIGINS`**, not only in `LOCAL_DEV_ORIGINS`.

Restart `npm run dev:api` after CORS changes.

If `returnUrl` is missing or not allowlisted, login falls back to `VITE_DEFAULT_APP_URL` (HRMS on `5173`).

## 3. Backend of your app

### Install `@relproc/auth`

GitHub Packages, not npmjs. In the **product** repo root `.npmrc`:

```
@relproc:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

```bash
npm login --scope=@relproc --registry=https://npm.pkg.github.com
npm install @relproc/auth
```

Current version: `0.1.0`. Frontends do **not** install this package.

### Local secrets (`.dev.vars`)

Must be **byte-identical** to Auth `apps/api/.dev.vars` for the Auth process you are hitting:

```
DATABASE_URL=          # YOUR product database, not Auth Neon
JWT_SECRET=            # copy from Auth
INTERNAL_TOKEN=        # copy from Auth
AUTH_API_URL=http://localhost:8787
ENVIRONMENT=development
CORS_ORIGINS=
```

`ENVIRONMENT=development` lets your Worker add localhost origins in CORS. Keep `DATABASE_URL` as **your** app’s Neon.

### Gate routes

```ts
import { requireAuth, requirePermission, getAuth, can } from '@relproc/auth';

app.use('/api/*', requireAuth());

app.get('/api/records', requirePermission('qms:write'), (c) => {
  const auth = getAuth(c);
  return c.json({ id: auth.id, email: auth.email, ok: can(auth, 'qms:write') });
});
```

`requireAuth` verifies `relproc_session`, calls Auth `GET /internal/session/:userId` with `INTERNAL_TOKEN` (cached 60s), and sets `c.get('auth')`.

Your product CORS allowlist must include **your UI origin** (and typically Auth web is not a caller of your API).

## 4. Frontend of your app

`.env` / `.env.development`:

```
VITE_AUTH_URL=http://localhost:5174
VITE_AUTH_API_URL=http://localhost:8787
VITE_API_URL=http://localhost:8790
```

Redirect to Auth when there is no session. Always pass `returnUrl` so you come back to **your** app:

```ts
const AUTH_URL = import.meta.env.VITE_AUTH_URL;
const AUTH_API = import.meta.env.VITE_AUTH_API_URL;

export async function fetchSession() {
  const res = await fetch(`${AUTH_API}/auth/me`, { credentials: 'include' });
  if (!res.ok) {
    const returnUrl = encodeURIComponent(window.location.href);
    window.location.assign(`${AUTH_URL}/login?returnUrl=${returnUrl}`);
    return null;
  }
  return res.json() as Promise<{ user: SessionUser }>;
}

export async function logout() {
  await fetch(`${AUTH_API}/auth/logout`, { method: 'POST', credentials: 'include' });
  window.location.assign(`${AUTH_URL}/login`);
}
```

Gate UI with `user.permissions.includes('qms:write')`, not a hardcoded `role === 'Admin'`. `/auth/me` sends `primaryRole` (lowercase, e.g. `admin`) and a `role` alias of the same value.

Do not implement a login form in QMS/EDMS/HRMS.

## 5. Click-through test

1. Auth API + Auth web running.
2. Your API + your UI running.
3. Open your UI (e.g. `http://localhost:5178`).
4. Browser goes to `http://localhost:5174/login?returnUrl=http://localhost:5178/...`.
5. Sign in with a work account that already exists in Auth.
6. Browser returns to your UI; `GET /auth/me` is 200; your `GET /api/...` is 200 (not a red 200).

## 6. If it fails

| What you see | Cause |
| --- | --- |
| After login you land on HRMS `:5173` | `returnUrl` missing or origin not in `VITE_RETURN_ORIGINS`; restart Auth web |
| Network: red request, status 200, empty Preview | CORS. Your UI origin is not on the **API you called** (Auth `8787` for `/auth/me`, your API for `/api/*`) |
| 401 from your API after a good login | `JWT_SECRET` mismatch, or cookie sent to `127.0.0.1` while login was on `localhost` |
| 401 on `/internal/session` in Auth logs | `INTERNAL_TOKEN` mismatch, or `AUTH_API_URL` not `http://localhost:8787` |
| 403 `Forbidden: Requires …` | Session is valid; that permission is not on the user. Check `/auth/me` → `permissions` |
| Cookie never set | Opened `https://auth.relproc.com` from a local UI, or mixed `localhost` / `127.0.0.1` |

## Permissions

Authorization is the compiled list on the Auth user (`role grants ∪ extra grants`). Existing catalog includes `qms:write`, `qms:admin`, `edms:access`, `edms:admin`, `proposals:read|write|approve`, plus HRMS actions.

Need a **new** action? Add it in **this repo** (`packages/auth` permission catalog + seed), republish `@relproc/auth` if backends need the new type, and grant it on the user/role in Auth. Product apps do not keep a second permission table.

## Production (not this file)

When you deploy the product app:

- Frontend: `VITE_AUTH_URL=https://auth.relproc.com`, `VITE_AUTH_API_URL=https://auth-api.relproc.com`
- Backend secrets: **production** `JWT_SECRET` and `INTERNAL_TOKEN` (same as Auth prod), `AUTH_API_URL=https://auth-api.relproc.com`
- Ask Vinoth to add your production origin to Auth `CORS_ORIGINS` and Auth web `VITE_RETURN_ORIGINS` if it is not already there (`https://qms.relproc.com`, …)

Do not put `SEED_ADMIN_PASSWORD` or Auth `DATABASE_URL` in the product Worker.
