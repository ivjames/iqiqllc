# Deploying IQIQ LLC

Target: **https://iqiq.llc** — served from the lab980 droplet (conventions in
the `ivjames/lab980.com` repo's `CLAUDE.md`).

The site is static files with no dependencies: no build step, no app process,
no port, no pm2, no database. nginx serving the git checkout *is* the
deployment. Everything is driven by the operate CLI at `bin/iqiq`.

It is an **apex** site (`iqiq.llc` and `www.iqiq.llc`, not a `*.lab980.com`
subdomain), so two things come before the bring-up below and are outside this
repo's reach:

1. The registrar for `iqiq.llc` must delegate to DigitalOcean's nameservers
   (`ns1.digitalocean.com`, `ns2`, `ns3`).
2. The zone must exist in DigitalOcean: `doctl compute domain create iqiq.llc`.

Skip either and `iqiq setup` fails at DNS (no such zone) or at certbot (the
name never resolves to the droplet). `--no-dns` is the escape hatch if the zone
is managed elsewhere; create `A @` and `A www` -> `165.22.128.19` there first.

> Why not `provision-site`? That script scaffolds proxy-shaped sites (an app on
> a local port). This site has no app, so `iqiq setup` writes its own
> static vhost instead — same DNS/doctl, security-headers and certbot shape.

## One-time bring-up (on the droplet, as root)

```bash
git clone https://github.com/ivjames/iqiqllc /var/www/iqiq
ln -sf /var/www/iqiq/bin/iqiq /usr/local/bin/iqiq
iqiq setup
```

`iqiq setup` is idempotent and does, in order:

1. **DNS** — `doctl` A records `@` and `www` in zone `iqiq.llc` -> droplet IP
   (each skipped if it already exists; `--no-dns` to skip, `--ip` to override
   autodetect, `--no-www` to leave the alias out).
2. **nginx** — static vhost from `deploy/nginx.conf.template` installed as
   `/etc/nginx/sites-available/iqiq.llc`, symlinked into `sites-enabled/`,
   `nginx -t` + reload. One server block names `iqiq.llc www.iqiq.llc`. Root
   is the checkout; `index.html` is served `Cache-Control: no-cache` so
   deploys are live on the next visit; dotfiles (except `/.well-known`) and
   `*.md` are denied. An existing vhost is left untouched (certbot owns it
   after TLS).
3. **TLS** — waits for both names to resolve, then
   `certbot --nginx -d iqiq.llc -d www.iqiq.llc --redirect -n`: one cert
   spanning both. If DNS is still propagating it prints the exact certbot
   command to re-run.

## Deploying updates

Land changes on `main` (via a PR — see `CLAUDE.md`), then on the droplet:

```bash
iqiq deploy
```

That is `git fetch` + `git reset --hard origin/main` of the checkout, plus a
`sed` that stamps the deployed commit into the page's `BUILD` constant if it
has one. No build, no restart, no reload.

## Check it

```bash
iqiq status              # HEAD commit, apex + www probes, cert days + SAN list
health-check --site iqiq # the droplet-wide auditor also covers it
```

`status` prints the cert's `subjectAltName` line on purpose: `notAfter` alone
cannot tell you whether the www name is covered. Expect
`DNS:iqiq.llc,DNS:www.iqiq.llc`. Do not record a cert fact from an agent
session's own `openssl s_client` — the egress proxy re-signs those connections
and reports its own certificate (see the vinaigresque entry in lab980's
`CLAUDE.md`); read it on the box.

## Overrides

- `IQIQ_FQDN` — serve under a different name (default `iqiq.llc`)
- `IQIQ_BRANCH` — deploy a different branch (default `main`)
- `IQIQ_WWW=0` — drop the www alias (DNS, `server_name`, certbot) — same as
  `setup --no-www`
- `IQIQ_ZONE` / `IQIQ_RECORD` — override the DNS zone/record split (default
  zone `iqiq.llc`, record `@`)
