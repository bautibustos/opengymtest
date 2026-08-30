# Deploying openGym to Railway

This branch (`railway`) carries the changes needed to run openGym as **two Railway
services** instead of one `docker compose` host. `main` stays the plain self-hosted app.

## What's different from `main`

| File | Change | Why |
|---|---|---|
| `web/nginx.conf` → `web/nginx.conf.template` | `listen ${PORT}`, `resolver`, `proxy_pass` to `${API_UPSTREAM}` via a variable | Railway injects `$PORT`; the api is reached over private DNS (`api.railway.internal`), which only resolves at request time |
| `web/docker-entrypoint.d/05-resolver.envsh` | new — reads the nameserver from `/etc/resolv.conf` into `NGINX_RESOLVER` | works on Railway (IPv6 `fd12::` DNS) and under compose (`127.0.0.11`) unchanged |
| `web/Dockerfile` | extra stage clones `hasaneyldrm/exercises-dataset` and bakes `img/` + `gif/` into the image; installs the template | Railway can't share a volume between a sidecar and `web` |
| `docker-compose.yml` | drops the `media` service + shared mounts; sets `PORT`/`API_UPSTREAM` on `web` | keep local `docker compose up --build` working with the baked-media image |

The api service is **unchanged** — `api/Dockerfile` already listens on `process.env.PORT`
and stores everything under `DATA_DIR`.

## Services

```
web  (public)   build: web/Dockerfile        root dir: /       domain: xxx.up.railway.app
 └─ /api/*  ──private──▶  api  (no domain)    build: api/Dockerfile   root dir: /api
                          └─ volume mounted at /data
```

## Steps

1. **Push this branch** to GitHub (`git push -u origin railway`).

2. **Railway → New Project → Deploy from GitHub repo** → pick the repo.
   Set the deployed branch to `railway` (Service → Settings → Source).

3. **web service** (the one Railway creates first):
   - Settings → Source → **Root Directory**: `/`
   - Settings → Build → **Dockerfile Path**: `web/Dockerfile`
   - Settings → Networking → **Generate Domain**. Note it, e.g. `opengym-production.up.railway.app`.
   - Variables:
     ```
     PORT=8080
     API_UPSTREAM=api.railway.internal:3000
     ```

4. **api service**: project → **+ New → GitHub Repo** (same repo) or **Empty Service → connect repo**:
   - Settings → Source → **Root Directory**: `/api`   (uses `api/Dockerfile` on its own)
   - Settings → deployed branch: `railway`
   - **Do not** generate a domain — it stays private.
   - Add a **Volume**: mount path `/data`.
   - Variables (fill the domain from step 3):
     ```
     PORT=3000
     DATA_DIR=/data
     RP_ID=opengym-production.up.railway.app
     ORIGIN=https://opengym-production.up.railway.app
     RP_NAME=openGym
     # optional:
     # ADMIN_UIDS=<your user id from /data/db.json after first signup>
     # INVITE_ONLY=1
     # SESSION_DAYS=90
     ```

5. **Deploy both.** Open `https://<web domain>` and register a passkey.
   VAPID keys and the session secret are generated on first run into the `/data` volume,
   so they survive redeploys.

## Notes

- `RP_ID` / `ORIGIN` must match the web domain exactly. If you later add a custom domain,
  update both api variables and redeploy the api service — existing passkeys are bound to
  the old `RP_ID` and won't work on the new host.
- Making yourself admin: sign up first, open the api service's volume / logs to find your
  id in `db.json` (`users[].id`), set `ADMIN_UIDS`, redeploy api.
- Private networking is IPv6-only on Railway; the api listening via Node's default
  (`0.0.0.0`) also binds `::` on Railway's runtime, so `api.railway.internal:3000` works.
- Local sanity check of this branch: `cp .env.example .env && docker compose up --build`,
  then `http://localhost:8080` (media is now baked into the image, first build is slower).
