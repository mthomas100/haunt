# Deploy

Host: **Fly.io**. One Node machine, WebSockets, a 1 GB volume for `data/`, HTTPS out of the box.
Why Fly: the Dockerfile and `fly.toml` are all it needs, Fly builds remotely (no local Docker), WebSockets and
persistent volumes work on the cheapest machine (shared-cpu-1x, about $2 to 3 a month), and the whole deploy is one
command. Railway works too (`railway up`). Free tiers that sleep or have no disk are out for a live event.

## 5-minute path

```bash
brew install flyctl                      # once
fly auth login
cp .env.example .env                     # optional: fill in what you want (sign-in providers, ADMIN_TOKEN, ...)
# edit fly.toml: set `app` to a name of your own (the app name is the URL: https://<app>.fly.dev)
scripts/deploy.sh                        # creates app + volume, stages .env as Fly secrets, deploys, smoke-checks
```

Then: `scripts/deploy.sh logs`, `scripts/deploy.sh status`, `scripts/deploy.sh rollback`.
Set `PUBLIC_BASE_URL` to the final `https://<app>.fly.dev`: OAuth callbacks are built from it.
Change a single secret later with `fly secrets set KEY=value -a <app>`.

## CI

`.github/workflows/ci.yml` runs on every push and PR: `npm ci`, `node --check server/*.js`, and `npm test`, the
smoke test that boots the server and exercises the API, the WebSocket pushes, consent and the kill switch.

To deploy on every push to `main`, add a job that runs `flyctl deploy --remote-only` with a `FLY_API_TOKEN`
repo secret (`fly tokens create deploy -a <app>`).

## Base44 mirror (optional second frontend host)

The same `web/` pages can also be served from Base44 hosting, calling the Fly API and socket, so a phone on either
URL hunts and haunts phones on the other. The team ran one at https://aura-haunt.base44.app.

- **Build:** `node scripts/base44.mjs build` writes `dist/base44/`: the FaceX weights, `window.HAUNT_API` (the Fly
  URL) in every page, a router in `index.html` (Base44 answers any path that is not a file with `index.html`, so
  `/register` goes to `/register.html`), and `?v=<sha>` on every local script and stylesheet, because Base44 caches
  those for an hour.
- **Deploy:** `BASE44_APP_ID=<your app id> node scripts/base44.mjs deploy` builds, uploads to Base44's deploy-dist
  API, waits for the new build and runs live checks. It needs `BASE44_TOKEN` (a Full-access personal access token
  for that app) or your own Base44 CLI login (`npx base44 login`, once). Limits: 50 MB, 1,000 files, 5 deploys a
  minute per workspace.
- **Fly side:** `FRONTEND_ORIGINS` in `fly.toml` lists the origins allowed to call the API and to receive sign-in
  handoffs (`server/cors.js`). Add the mirror's origin there first.
- **Sign-in on the mirror:** the button goes to Fly's `/auth/<provider>?return=<mirror page>`; the callback returns
  to that page with `#haunt_session=...`, and the page sends it as a header. A third-party cookie would not work,
  because iOS Safari drops them.

Nothing about faces goes to Base44: matching runs on the phone and the gallery stays on Fly.

## No account yet? Tunnel fallback

`npm start` plus `npm run tunnel` (cloudflared quick tunnel) gives a public HTTPS URL to the laptop in 10 seconds.
Phones need HTTPS for the camera. The URL changes on every restart and dies with the laptop, so use it for phone
tests, not for the event.

## What runs where

- One machine, `PORT=3000`, `DATA_DIR=/data` (the volume). The store is `/data/db.json`; delete it after the event.
- FaceX weights are baked into the image at build time (Dockerfile), so cold starts download nothing.
- Secrets reach the app as Fly secrets (`fly secrets list -a <app>`), never as files in the image or the repo.
