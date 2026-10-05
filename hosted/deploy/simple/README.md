# Static deployment

This deployment runs one Waku container per preallocated account. Caddy asks
for a username and password before forwarding any dashboard request. Each
account has a separate hostname, data directory and Docker network. The
deployment needs Docker Compose and a DNS A record for every hostname. It
uses the existing tenant image and remains under hosted/LICENSE (Elastic
License 2.0).

This deployment does not use Supabase, the spawner, XFS quotas, automatic idle
shutdown or the metering proxy. Containers remain running, and their data can
fill the host disk. Operators manage accounts and backups themselves.

## Build and configure

Build the image from a clean checkout:

```bash
bash hosted/image/build.sh --tenant-only --tenant-tag waku-simple:local
```

Create an accounts JSON file outside the checkout. Each account needs
`username`, `hostname` and `password_hash` (bcrypt). Caddy's `hash-password`
command creates a bcrypt hash; run it interactively rather than passing a
password as a command-line argument. Assign the apex hostname to the first
account and distinct subdomains to subsequent accounts. Do not commit hashes
or passwords.

```bash
docker run --rm -it caddy:2.10.2-alpine caddy hash-password
python3 hosted/deploy/simple/render.py /srv/waku-simple/accounts.json \
  --output /srv/waku-simple
```

For each username, create `/srv/waku-simple/tenants/<username>/home` and
`env`, owned by UID/GID `10001:10001`. Create an empty `env/.env` at mode
0600 and a `home/mcp.json` containing `{"servers": []}`. Preserve existing
files when adding an account. Waku writes its default SOUL and local memory
on startup. Each member configures their own model key in the dashboard.

```bash
cd /srv/waku-simple
docker compose -f compose.json config --quiet
docker compose -f compose.json up -d
```

Caddy obtains individual HTTPS certificates using ports 80 and 443; it does
not require a wildcard certificate or DNS API token. The generated Caddyfile
protects all paths, strips the browser's Authorization header before forwarding
requests, and disables response buffering for streaming chat. Tenant ports
are not published on the host.

If website-port validation fails, obtain a certificate with a DNS challenge
and set `"tls": "manual"` on the relevant account. Install the certificate
chain at `/srv/waku-simple/tls/<username>/fullchain.pem` and its private key
at `privkey.pem` beside it. Keep the key root-owned at mode 0600. Render again
and recreate Caddy with `docker compose up -d caddy`. Only Caddy receives the
certificate directory, mounted read-only; tenant containers do not receive it.
Caddy does not renew manually supplied certificates. Operators must renew
them before their expiry, replace both files and reload Caddy. A manual DNS
challenge needs another DNS update at renewal unless a DNS API automates it.

## Add or disable an account

To add an account, add its DNS record and JSON entry, prepare its directories,
render both configuration files again, then run `docker compose up -d` and
`docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`.

To disable an account, remove its JSON entry, render again, reload Caddy and
run `docker compose up -d --remove-orphans`. Preserve its data directories.
Browser password prompts do not provide application sessions or a logout
button; users should close the browser session on shared machines. Rotating
an account's bcrypt hash and reloading Caddy invalidates its old password.

## Verify

Confirm that unauthenticated requests to the homepage and `/api/data` return
401 over HTTPS, that valid credentials reach the setup page, and that a
second account's credentials cannot reach the first account's hostname.
Restart a tenant and confirm that its saved conversation is still available.
Use `docker compose ps` and `docker compose logs` to diagnose startup errors.

## Develop the laboratory fork

The fork keeps `main` aligned with `upstream/main`. The long-lived `lab` branch
holds laboratory changes, including this static deployment. Feature branches
start from `lab`, and their pull requests target `lab`. Keep the GitHub default
branch as `main`, but select `lab` explicitly as the base of laboratory PRs.

Start a feature from the latest laboratory version:

```bash
git fetch origin
git switch lab
git pull --ff-only origin lab
git switch -c feature/knowledge-access
```

Commit the feature, push its branch, and open a PR with base `lab`:

```bash
git push -u origin feature/knowledge-access
```

GitHub runs the validate and hosted-docker workflows on PRs, including PRs
targeting `lab`. Review the changes and checks before merging. The hosted Docker
checks require a Docker daemon and XFS; the deterministic suite runs offline.

Use GitHub's Sync fork on `main` when the original repository changes. Bring
those updates into `lab` through a separate PR:

```bash
git fetch origin
git switch main
git pull --ff-only origin main
git switch lab
git pull --ff-only origin lab
git switch -c sync/lab-upstream-YYYYMMDD
git merge origin/main
```

If Git reports conflicts, combine the intended behavior, remove conflict
markers, stage the resolved files and commit the merge. Run the relevant
deterministic evals and lint, push the sync branch, then open its PR against
`lab`. Preserve the shared `lab` history with merges rather than rebasing it.

Production releases select a tested commit from `lab`. Pushing or merging a
branch does not upgrade the running laboratory server. Waku does not yet have
the isolated staging and manual release Actions configured for fast-llm.
The `lab-deployment-2026-10-04` tag records the earlier production source
baseline; `deploy/lab-baseline`, `deploy/lab-pr` and the earlier upstream sync
branch remain available as historical references. New work belongs on feature
branches from `lab`.
