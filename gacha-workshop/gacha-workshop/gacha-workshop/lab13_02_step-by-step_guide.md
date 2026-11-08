# Lab 13 — Task 02 step-by-step (Local Nginx Proxy Manager)

Your identity: **STUDENT_ID 6788185** → Compose project `gacha-6788185` → volume `gacha-6788185_pgdata`
Your machine: Windows + Docker Desktop (linux-desktop), PowerShell
Your original (experiment 01) row: **draw id 1**, `Copper Golem` / `COMMON`, `2026-11-06T03:54:09.351Z`

Open PowerShell **inside the project folder** (the one that contains `compose.yml` and `.env`).
Every command below is written to run from that folder, one at a time, in order.

---

## Step 0 — Pre-flight checks (2 min)

```powershell
docker version --format '{{.Server.Version}}'   # must print a version, e.g. 28.x
docker compose version
Get-Content .env                                 # STUDENT_ID=6788185 must be there
```

- If `docker version` says *cannot connect / pipe not found*, start **Docker Desktop** and wait for the whale icon to stop animating.
- Port 80 and 443 must be free for NPM. Check first:

```powershell
Get-NetTCPConnection -State Listen -LocalPort 80,443 -ErrorAction SilentlyContinue
```

Empty output = free. If something is listening, stop it before continuing (a leftover lab12 nginx container is the usual culprit — `docker ps` to identify).

Why this matters: `compose.proxy.yml` publishes `80:80` and `443:443`. If another process owns them, `up` fails with *port is already allocated*.

---

## Step 1 — Stop the local stack **without `-v`**

```powershell
docker compose --env-file .env -f compose.yml -f compose.local.yml ps
docker compose --env-file .env -f compose.yml -f compose.local.yml down
```

Expected: `Container gacha-6788185-app-1 Removed`, `Container gacha-6788185-db-1 Removed`, `Network gacha-6788185_default Removed` — and **no** mention of the volume.

Then prove the volume survived:

```powershell
docker volume ls --filter name=gacha-6788185
docker volume inspect gacha-6788185_pgdata --format '{{.Name}} created {{.CreatedAt}}'
```

`CreatedAt` is still the date from experiment 01 → the volume was kept, not recreated.

> **Never** use `down -v` here. `-v` deletes `gacha-6788185_pgdata` and with it the draw id 1 you must show later.

---

## Step 2 — Start the same project with production + proxy

```powershell
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml up -d --build --wait
```

Expected: `Image gacha-6788185-app Built`, `Container gacha-6788185-db-1 Healthy`, `Container gacha-6788185-app-1 Healthy`, `Container gacha-6788185-npm-1 Started`.

What each file contributes (be ready to say this):

| File | Adds |
| --- | --- |
| `compose.yml` | base: app (build, `PORT=3000`, PG\* vars, healthcheck) + db (`postgres:18-bookworm`, volume `pgdata`) — **no `ports:` at all** |
| `compose.production.yml` | only `restart: unless-stopped` for app and db |
| `compose.proxy.yml` | `npm` service (`jc21/nginx-proxy-manager:2.16.0`) publishing `80`, `443` and `127.0.0.1:81`, with its own `npm_data` / `npm_certs` volumes |

`compose.local.yml` is **not** in the list, so `127.0.0.1:8085:3000` no longer exists. That is exactly the point: the app has no host port any more.

**Screenshot 02-1** now: the `down` output and the `up --build --wait` output in one terminal window.

---

## Step 3 — Confirm the topology

```powershell
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml ps
```

Expected shape (ports column):

```
NAME                  SERVICE   STATUS              PORTS
gacha-6788185-app-1   app       Up (healthy)        3000/tcp            <- no host port
gacha-6788185-db-1    db        Up (healthy)        5432/tcp            <- no host port
gacha-6788185-npm-1   npm       Up                  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:81->81/tcp
```

Shared network (this is what makes `app:3000` resolvable):

```powershell
docker network inspect gacha-6788185_default --format '{{range .Containers}}{{.Name}} {{end}}'
```

Expected: `gacha-6788185-app-1 gacha-6788185-npm-1` (db joins as well; app and npm are the two that matter here).

Optional, the exact upstream proof:

```powershell
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml exec npm getent hosts app
```

**Screenshot 02-2** now: `ps` output + the `network inspect` line (one screenshot, two commands visible).

---

## Step 4 — NPM admin, first-run setup

Browser: **<http://127.0.0.1:81>**

- First visit shows the *Initial Administrator Setup* form (older docs call the default login `admin@example.com` / `changeme`; with 2.x you set your own email + password on this screen).
- Enter your own email and a password **you** keep. Log in.
- Port 81 is published on `127.0.0.1` only, so nothing outside your laptop can reach the admin UI — that is deliberate, and it is what you will tunnel to on the Ubuntu VM in task 03.

No screenshot is required for the login screen; **never** screenshot a password.

---

## Step 5 — Add the hosts entry (keeps existing entries)

PowerShell **as Administrator** (Start → right-click Windows PowerShell → Run as administrator):

```powershell
Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value "`r`n127.0.0.1 gacha.test"
```

`Add-Content` **appends**; it never rewrites the file, so existing entries stay. Verify:

```powershell
Select-String -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Pattern "gacha.test"
ipconfig /flushdns
Resolve-DnsName gacha.test | Select-Object Name,IPAddress
```

Expected: `127.0.0.1 gacha.test` and `Resolve-DnsName` returns `127.0.0.1`.

Why the hosts file: `.test` names do not exist in public DNS; the laptop's hosts file is the smallest possible stand-in for a DNS record (task 03/04 replace it with real DNS/HTTPS).

---

## Step 6 — Create the Proxy Host

NPM → **Hosts → Proxy Hosts → Add Proxy Host**

| Field | Value |
| --- | --- |
| Domain Names | `gacha.test` |
| Scheme | `http` |
| Forward Hostname / IP | `app` |
| Forward Port | `3000` |
| Block Common Exploits | ✔ on |
| Websockets Support | ✔ on (harmless, helps nothing here but is the usual default) |
| Access List | Public / none |
| SSL tab | **None** — a `.test` name cannot get a public certificate; this task uses local HTTP |

Save. Do **not** put `localhost` or `127.0.0.1` in the Forward Hostname field.

> **Why:** `localhost` inside a container is that container's own loopback. In the `npm` container nothing listens on 3000, so NPM would answer **502 Bad Gateway**. `app` is the service name on the shared network, which Docker's embedded DNS resolves to the app container's IP; `3000` is the port the app listens on (`PORT=3000`).

**Screenshot 02-3** now: the proxy host row/details with `gacha.test`, `http`, `app`, `3000`.

---

## Step 7 — Add the `/backend/` Custom Location

On the same proxy host: **Edit → Custom locations → Add location**

| Field | Value |
| --- | --- |
| Location | `/backend/` |
| Scheme | `http` |
| Forward Hostname / IP | `app` |
| Forward Port | `3000` |

Then click the **gear (⚙)** on that location row and paste **only** this into the advanced Nginx box:

```nginx
rewrite ^/backend/(.*)$ /$1 break;
```

Save the location, then Save the proxy host.

Rules for that box:

- Put **nothing else** — no `location /backend/ { }` wrapper, no `proxy_pass`. NPM generates the location block and the `proxy_pass http://app:3000;` itself; duplicating them produces a broken or duplicated upstream.
- The `break` flag means "rewrite, then continue with the (NPM-generated) `proxy_pass`", and `$1` strips the `/backend` prefix: `/backend/test` → `/test`, `/backend/api/draw` → `/api/draw`.
- Keep the trailing slash in the Location field: `/backend/`, not `/backend`.

**Screenshot 02-4** now: the location form **and** the advanced field with the single `rewrite` line visible.

---

## Step 8 — Test through the proxy

Open **<http://gacha.test/backend/test>**

Press, in this order (the page suggests the same order):

1. `GET · Health → DB` → **200** `{"status":"ok","database":"connected","app":"gacha"}`
2. `GET · Wallet` (Student ID `6788185`) → **200**, wallet JSON
3. `POST · Recharge` 1000 → **201**
4. `POST · Exchange` 1000 → **201**
5. `POST · Draw once` → **201**, a draw result (this may be draw id 2 — that is fine, it is a new button test, not the experiment-01 evidence)
6. `GET · Saved history` → **200** with `transactions` and `draws`

What to check in the evidence panel: the request URL shows `http://gacha.test/backend/api/...`, and the status is 200/201 — that is the local page deriving its prefix (`/backend`) and the rewrite stripping it.

**Screenshot 02-5** now: the page with a 200/201 status and the JSON response visible (health + one successful POST is enough in one frame).

---

## Step 9 — Prove the original row survived

With `GET · Saved history` (do not draw again just to produce evidence):

- Find **`"id": "1"`** in `draws` → `"result_name": "Copper Golem"`, `"rarity": "COMMON"`, `"transaction_id": "3"`, `"created_at": "2026-11-06T03:54:09.351Z"`.
- That is the identical row from experiment 01. Nothing was re-inserted; only the containers and the request path changed.

Speak this sentence in the room: *"`down` without `-v` kept the named volume `gacha-6788185_pgdata`; STUDENT_ID stayed 6788185, so the project name — and therefore the volume — stayed the same. That is why the same draw id 1 is still here after switching from the local port to NPM."*

Optional CLI cross-check (same data, no new row):

```powershell
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml exec db psql -U ball -d gacha -c "SELECT id, student_id, result_id, result_name, rarity, created_at FROM draws ORDER BY id;"
docker volume inspect gacha-6788185_pgdata --format '{{.Name}} created {{.CreatedAt}}'
```

**Screenshot 02-6** now: the saved history JSON showing `"id": "1"` and `"created_at": "2026-11-06T03:54:09.351Z"`.

---

## Step 10 — Fill the report (file `6788185_lab13.docx`, section 02 only)

The table and the request path are already filled for you. What is left is the six red
`[SNAPSHOT 02-x]` lines: replace each red paragraph with the matching screenshot
(Insert → Pictures → This Device), then delete the red text. Keep them in order:

| Placeholder | Screenshot |
| --- | --- |
| 02-1 | `down` + `up ... --build --wait` terminal output |
| 02-2 | `ps` (app without ports) + `network inspect` |
| 02-3 | NPM proxy host `gacha.test` → `app:3000` |
| 02-4 | `/backend/` custom location + `rewrite` line |
| 02-5 | `http://gacha.test/backend/test` with 200/201 buttons |
| 02-6 | saved history with draw id 1 and its `created_at` |

Sanitising: no passwords, no NPM admin password, no tokens. The `.env` values
(`POSTGRES_PASSWORD`) must not appear in any screenshot.

---

## Step 11 — Answers to have ready (in-room check)

1. **Which files did you select and why?** `compose.yml` + `compose.production.yml` + `compose.proxy.yml`. Explicit `-f` means `compose.override.yml` is *not* auto-loaded, and the local overlay is not chained in.
2. **Why does the app have no host port now?** The base file never declares `ports:` for `app`; only the local overlay did, and only NPM publishes host ports. Privacy by design: db and app are reachable only inside the project network.
3. **Why `app:3000` and not `localhost:3000`?** NPM's `localhost` is the NPM container's own loopback; `app` is a DNS name on the shared Docker network, and 3000 is the app's internal port.
4. **What does the rewrite do?** Strips `/backend` before the request reaches the app, so one app can serve several URL prefixes; NPM still owns the `location`/`proxy_pass`.
5. **Why did the data survive?** Named volume + unchanged project identity (`name: gacha-${STUDENT_ID}`). `down` keeps volumes; `down -v` would delete them.
6. **What would break it?** `-v` on `down`, changing `STUDENT_ID`, using `-p` with a different project name, or switching Docker host (VM/Railway — those are new storage, task 03/04).

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `up` fails: *port 80/443 already allocated* | another container or IIS owns the port | `Get-NetTCPConnection -State Listen -LocalPort 80`; stop that container/service, retry |
| `docker: cannot connect ... pipe` | Docker Desktop not running | start Docker Desktop, wait for *Engine running* |
| `gacha.test` does not resolve | hosts file not saved, or not admin | re-run `Add-Content` in an **admin** shell, `ipconfig /flushdns`, test with `Resolve-DnsName` |
| Browser silently opens `https://gacha.test` | an earlier HSTS/redirect entry | type `http://gacha.test/backend/test` explicitly, or clear site data for `gacha.test` |
| **502 Bad Gateway** | forward host is `localhost`/`127.0.0.1`, or wrong port | set `app` + `3000`; check app health with `docker compose ... ps` |
| **404 Route not found** at `/backend/test` | custom location not saved, or `/backend` without trailing slash, or rewrite in the wrong box | Location `/backend/`, rewrite in the location's ⚙ advanced field |
| Buttons hit `/api/...` and 404 | page opened as `/test` through a normal proxy host without the prefix | open `http://gacha.test/backend/test`; the page derives the prefix from its URL |
| NPM logs show *duplicate proxy_pass* / config error | pasted a full `location` block into the advanced field | keep only the single `rewrite` line |
| `image jc21/nginx-proxy-manager:2.16.0 not found` | tag changed upstream | `docker pull jc21/nginx-proxy-manager:latest` works, but do not edit `compose.proxy.yml` for the lab; report it to the LA |

---

## Cleanup (only after the LA has seen it)

```powershell
docker compose --env-file .env -f compose.yml -f compose.production.yml -f compose.proxy.yml down
```

No `-v` — task 03 will need the same volume if you reuse this laptop, and it keeps your evidence intact. Remove the hosts entry only if you no longer need the name.
