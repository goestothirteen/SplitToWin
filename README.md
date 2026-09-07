# SplitToWin — infrastructure

AI receipt bill-splitter. This repo holds the **deployment configuration**; the
application code lives in two others:

| Repo | What it is |
|---|---|
| [SplitToWin-ui](https://github.com/goestothirteen/SplitToWin-ui) | React frontend. Built to static files, served by Caddy. |
| [SplitToWin-api](https://github.com/goestothirteen/SplitToWin-api) | Flask API. Sends the photo to a model, returns line items. |

Live at **https://split2win.duckdns.org**

## What the app does

Photograph a restaurant receipt → the API sends the image to a model, which
returns structured line items with quantities, and classifies each line as food,
service charge, tax, discount, or rounding → name the people at the table → tap
each line to say who had it → it works out what everyone owes.

Service charge and tax are apportioned **in proportion to what each person ate**,
not split evenly, and all arithmetic is in integer cents with largest-remainder
rounding so the per-person amounts sum to the receipt total exactly.

## Production layout

One droplet, one shared reverse proxy, one origin per app.

```
                    ┌─────────────────────────────────────┐
  :443 ─────────────►  edge-caddy   (~/edge)              │
                    │  TLS + routing for the whole box    │
                    └──┬───────────────────────┬──────────┘
                       │                       │
   split2win.duckdns.org                fridgeoflove.duckdns.org
                       │                       │
        ┌──────────────┴───────────┐           └──► fridge:8025
        │                          │
   /api/*                         /*
        │                          │
        ▼                          ▼
  splittowin-api:8000      /srv/split2win  (static files, no container)
  gunicorn, 2×4 threads    built into ~/splittowin/dist
```

The UI and the API share **one hostname**. That is deliberate: the browser never
makes a cross-origin request, so there is no CORS to configure and the API
cannot be used as an open proxy from another site.

**Caddy owns the front door and lives in its own project.** It used to sit
inside the fridge project, which meant redeploying the fridge restarted the
proxy and briefly interrupted every other app. Apps now join the shared `edge`
network to become reachable.

## Server layout

```
~/edge/            docker-compose.yml, Caddyfile     — shared reverse proxy
~/splittowin/      docker-compose.yml, Makefile, .env
  ├── api/         clone of SplitToWin-api
  ├── ui/          clone of SplitToWin-ui
  └── dist/        built frontend, bind-mounted read-only into Caddy
~/fridge/          unrelated app, also on the edge network
~/fpl-bot/         unrelated app
```

## Deploying

From `~/splittowin` on the server:

```bash
make deploy
```

That pulls both repos, rebuilds the frontend and the API image, restarts the
API, and verifies the result. `make help` lists everything else.

| Command | What it does |
|---|---|
| `make deploy` | full deploy: pull, build both, restart, health-check |
| `make health` | check container health, the API through Caddy, and the frontend |
| `make logs` | follow API logs |
| `make stats` | who has used the app, one row per person (JSON of `/api/stats`) |
| `make versions` | show the deployed commit of each repo |
| `make rollback` | step both repos back one commit and redeploy |
| `make clean` | reclaim Docker build cache |

The frontend build takes ~90s on this box (1 vCPU). It builds into `dist.new`
and only then syncs into `dist/`, so a failed build never leaves half a site
being served. It **syncs contents rather than replacing the directory** —
Caddy bind-mounts `dist/`, and a bind mount follows the inode, so swapping the
directory would silently leave the proxy serving the old one.

## Rolling back

`make rollback` steps both repos back one commit and redeploys. Builds are
reproducible because every dependency is pinned, so the previous commit rebuilds
into the same thing it was before.

To go further back, or to move the repos independently:

```bash
git -C api log --oneline -10        # find the commit
git -C api reset --hard <sha>
make build-api up health
```

Nothing in the app is stateful — there is no database and no user data on the
server — so a rollback is only ever a code change. The one piece of state that
matters is Caddy's certificate store.

## Things that will bite you

**Do not delete the `fridge_caddy-data` volume.** It holds the Let's Encrypt
certificates for both sites. The name is a leftover from when Caddy lived in the
fridge project; `~/edge/docker-compose.yml` deliberately references it as an
external volume so the existing certificates survived the move. Deleting it
means re-issuing against Let's Encrypt's rate limit.

**DNS is DuckDNS.** Both hostnames point at this droplet's IP. If the IP ever
changes, update it at duckdns.org *first*, then restart Caddy.

**The `.env` is only on the server.** It holds the API key and is `chmod 600`.
It is not in any repo. If the droplet is lost, recreate it from
`splittowin/.env.example`.

**The box is 1 vCPU / 2 GB** shared with two other apps. The API is capped at
512 MB and the frontend build caps its heap at 768 MB, so neither can invoke the
OOM killer against a neighbour.

## Monitoring

`make health` is the manual check. The API container has a Docker health check
that polls `/healthz` every 30s — deliberately model-free, so it keeps answering
while a parse is in flight. `docker ps` shows the result.

`/healthz` returns which providers are actually usable, so a missing key or an
unavailable provider shows up as `degraded` there rather than as a mystery
failure at the dinner table.

## Who has used it

**https://split2win.duckdns.org/api/stats** — one row per person who parsed
a receipt: device, first and last seen, page views, receipts parsed, failed
parses, average parse time. A browser gets a table; `curl` (or `make stats`)
gets the JSON.

It is an open URL on purpose. The report is counts and device classes only —
no addresses — so there is nothing on it worth the friction of a login.

How it works: Caddy writes a JSON access log for this site to
`~/edge/logs/split2win.log` (rotated by Caddy, kept a year); the API
container mounts that directory read-only and summarises it on request,
cached for 30s. The log holds HTTP metadata only — address, time, path, user
agent. Pay-link URLs would carry a payee's PayNow number and names in the
path, so the Caddyfile blanks that part before it is written. Names, items
and assignments never reach the server at all.

Logging config lives in `~/edge/Caddyfile`, so changing it is an edge deploy:
`cd ~/edge && docker compose up -d --force-recreate caddy`. The mount into the
API is in `~/splittowin/docker-compose.yml`, picked up by `make up`.
