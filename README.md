# mailcow on Coolify

A self-contained snapshot of [mailcow-dockerized](https://github.com/mailcow/mailcow-dockerized)
(stock `docker-compose.yml` + full `data/` config tree) prepared to deploy as a
**Docker Compose application in Coolify**. mailcow has no official Coolify support —
this repo works around that by shipping everything the compose file bind-mounts.

> **Make this repository PRIVATE.** And put `coolify.env` values into Coolify's
> environment editor — never leave real secrets in a public repo.

## 1. Push to GitHub

```bash
unzip mailcow-coolify.zip && cd mailcow-coolify
git init -b main && git add -A && git commit -m "mailcow for coolify"
git remote add origin git@github.com:YOURNAME/mailcow-coolify.git
git push -u origin main
```

## 2. Create the app in Coolify

1. **+ New Resource → Docker Compose** (from your Git repository), branch `main`.
   Use a **fresh resource** — don't reuse a previously failed mailcow app
   (its storage dir contains config paths wrongly created as directories).
2. In the app's settings enable **“Preserve Repository During Deployment”**.
   This copies the repo (incl. `data/`) into `/data/coolify/applications/<uuid>/`
   so all single-file bind mounts (unbound.conf, redis-conf.sh, …) resolve.
3. Open **Environment Variables → Developer view** and paste the entire
   contents of `coolify.env`. Edit the four `CHANGE` values:
   - `MAILCOW_HOSTNAME` — e.g. `mail.yourdomain.com`
   - `TZ` — your timezone
   - `COMPOSE_PROJECT_NAME` — **the Coolify app UUID** (the random string in the
     app's URL and under `/data/coolify/applications/`). Required so mailcow's
     cron (ofelia), watchdog and UI container-controls can find the containers.
   - ACME DNS provider + API credential (see Certificates below)
4. **Deploy.**

## 3. After deploy

- Web UI / webmail: `https://mail.yourdomain.com:8443` — login `admin` / `moohoo`
  → change the password immediately.
- Open firewall ports: `25, 465, 587, 143, 993, 110, 995, 4190` (and `8443` for the UI).
  Ask your provider to unblock **outbound port 25**.
- DNS: `A mail.yourdomain.com`, `MX yourdomain.com → mail.yourdomain.com`,
  **PTR/rDNS of the server IP = mail.yourdomain.com**, SPF, DMARC, and the DKIM
  key mailcow generates (UI → Configuration → ARC/DKIM keys).
  Optional: `autodiscover` / `autoconfig` CNAMEs → `mail.yourdomain.com`.

## Certificates (why DNS-01)

Coolify's proxy occupies ports 80/443, so mailcow can't do the normal HTTP-01
Let's Encrypt flow. `coolify.env` therefore enables **DNS-01**
(`ACME_DNS_CHALLENGE=y`): mailcow's acme container creates a DNS TXT record via
your DNS provider's API (any acme.sh `dns_*` provider — Cloudflare = `dns_cf` +
`CF_Token`). Postfix/Dovecot/nginx then get real certificates automatically.
Docs: https://docs.mailcow.email/post_installation/firststeps-ssl-dns/

No API at your DNS provider? Move the domain's DNS to one that has an API
(e.g. Cloudflare), or route `http://mail.yourdomain.com:8080` through Coolify's
proxy and set `ACME_DNS_CHALLENGE=n` so HTTP-01 validation can pass through.

## Optional: clean URL through Coolify's proxy

To serve the UI on standard 443 too: enable **Connect To Predefined Network**,
set the domain `https://mail.yourdomain.com:8080` on the `nginx-mailcow`
service, and add `TRUSTED_PROXIES=<coolify proxy subnet>` to the env.
Mail clients keep using the direct ports regardless.

## Updating mailcow

`update.sh` does not work here. Instead sync this repo with upstream and redeploy:

```bash
git remote add upstream https://github.com/mailcow/mailcow-dockerized.git
git fetch upstream && git merge upstream/master   # keep your side for conflicts in data/
git push
```

Then hit **Redeploy** in Coolify — containers migrate the DB automatically.

## Important caveats

- **Never delete the resource in Coolify** — mail, DB and Redis live in named
  Docker volumes that are removed with it. Back up first
  (`helper-scripts/backup_and_restore.sh` from upstream works if you export the
  same env vars, incl. `COMPOSE_PROJECT_NAME`).
- “Preserve Repository” re-copies repo files on every deploy: local edits on the
  server to files that exist in the repo are reset (extra files like
  `data/conf/postfix/extra.cf` survive). Make config changes in this repo instead.
- If a deploy ever fails with *“not a directory”*: a file mount was pre-created
  as a directory. SSH in and `rm -rf /data/coolify/applications/<uuid>/data`,
  then redeploy.
- `netfilter-mailcow` (fail2ban) runs with `network_mode: host` + privileged —
  that's stock mailcow behaviour.
