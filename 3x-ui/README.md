# Naive behind 3x-ui Pro

Companion installer that adds [NaïveProxy](https://github.com/klzgrad/naiveproxy) + [Caddy](https://caddyserver.com/) to a server **already running [3x-ui Pro](https://github.com/mozaroc/x-ui-pro)**, without fighting over `:443`.

The vanilla [`../setup.sh`](../setup.sh) wants exclusive ownership of `:443` and `:80`. 3x-ui Pro already binds both (nginx SNI router on `:443`, redirect server on `:80`). This script plugs Caddy into the existing nginx as a third SNI upstream on loopback instead.

## Run

```bash
wget -qO- https://raw.githubusercontent.com/Ieveltyanna/naive-caddy-installer/main/3x-ui/setup.sh | sudo bash
```

Non-interactive:

```bash
wget -qO- https://raw.githubusercontent.com/Ieveltyanna/naive-caddy-installer/main/3x-ui/setup.sh | \
  sudo NAIVE_DOMAIN=naive.example.com NAIVE_MASK_SITE=https://www.lovense.com bash
```

Optional: `NAIVE_BACKEND_PORT=8444` (default) — change only if `:8444` on loopback is taken.

> **Pick your own mask site.** The default `https://www.lovense.com` is fine for a one-off install, but if you take it as-is across many VPSes the cover-site choice itself becomes a fingerprint. Run [RealiTLScanner](https://github.com/XTLS/RealiTLScanner) from a different machine to find a site whose TLS fingerprint matches your VPS's IP-subnet neighbours. See the «Picking a mask site» section in [`../README.md`](../README.md#picking-a-mask-site).

## Prerequisites

- 3x-ui Pro is **already installed** and working. Both certs (`/etc/letsencrypt/live/<panel>/`, `/etc/letsencrypt/live/<reality>/`) must exist.
- A **third domain** for Naive — must differ from the panel domain and the reality domain. DNS A-record points at this VPS, no Cloudflare orange-cloud.
- nginx (with `stream` and `ssl_preread`) is the same one 3x-ui Pro installed.

The script aborts with a clear message if 3x-ui Pro isn't detected (`/etc/nginx/stream-enabled/stream.conf` missing or doesn't have `ssl_preread`/`proxy_protocol on`).

## Architecture

```
public                                                   loopback
─────                                                    ────────
                       ┌─ map $ssl_preread_server_name ─┐
                       │   <reality>      → xray   ─────┼──→ 127.0.0.1:8443  (Reality / xray)
                       │   <panel>        → www    ─────┼──→ 127.0.0.1:7443  (panel)
:443/tcp ─→ nginx ─────┤   <naive>        → naive  ─────┼──→ 127.0.0.1:8444  (Caddy + Naive, this script)
                       │   default        → xray   ─────┘
                       │
                       └─ proxy_protocol on (PROXY v1 to upstreams)

:80/tcp  ─→ nginx ─→ /etc/nginx/conf.d/naive-acme.conf  (HTTP-01 webroot for Naive cert)
                  └→ /etc/nginx/sites-available/80.conf (3x-ui Pro's redirect for panel/reality)
```

Caddy listens **only** on `127.0.0.1:8444`, accepts PROXY protocol v1 from nginx, terminates TLS using the cert at `/etc/letsencrypt/live/<naive_domain>/`, then runs `forward_proxy` (with the cover-site `reverse_proxy` for non-authenticated probes).

## What it changes on disk

| Path | Purpose | Survives 3x-ui Pro re-run? |
|---|---|---|
| `/etc/nginx/stream-enabled/stream.conf` | adds third arm (`<naive_domain> → naive`) and `upstream naive { 127.0.0.1:8444 }`. Existing `xray`/`www`/`default xray` arms preserved | **No** — re-run with `NAIVE_REPAIR=1` |
| `/etc/nginx/conf.d/naive-acme.conf` | `:80` server for the Naive domain — serves HTTP-01 challenges via webroot, redirects everything else to HTTPS | yes |
| `/etc/caddy/Caddyfile` | `auto_https off`, listener wrappers (`proxy_protocol` + `tls`), cert from `letsencrypt/live/<naive_domain>` | yes |
| `/etc/systemd/system/caddy.service` | systemd unit, `After=nginx.service` | yes |
| `/etc/letsencrypt/renewal-hooks/deploy/naive-caddy.sh` | reload Caddy after renewal (filters by `RENEWED_LINEAGE` so it only fires for the Naive cert) | yes |
| `/etc/cron.d/naive-cert-renew` | daily `certbot renew --cert-name <naive_domain> --webroot ...`. **Not** in root's user crontab — 3x-ui Pro greps that crontab and removes any line matching `certbot\|x-ui\|cloudflareips` | yes |
| `/etc/naive-proxy/state.env` | parameters for repair mode (domain, panel/reality, backend port, mask site) | yes |
| `/var/www/acme/` | webroot for HTTP-01 challenges | yes |

It does **not** touch:
- 3x-ui Pro's per-domain server configs (`sites-available/<panel>`, `<reality>`), `snippets/includes.conf`, the `:80` redirect, `nginx.conf`
- the panel SQLite database (`/etc/x-ui/x-ui.db`)
- existing Reality/Trojan/WS/XHTTP inbounds
- 3x-ui Pro's certs or its cron entries
- root's user crontab
- ufw rules (`:80` and `:443` are already open from 3x-ui Pro; backend port stays on loopback)

## Surviving 3x-ui Pro re-runs

`x-ui-pro.sh` wipes `/etc/nginx/sites-{available,enabled}/*` and `/etc/nginx/stream-enabled/*` on **every** invocation (lines 18–20 of x-ui-pro.sh, even without `-uninstall`). It also runs `fuser -k 80/tcp 443/tcp` (line 172) — but that doesn't reach Caddy on loopback.

After any re-run of x-ui-pro.sh:

```bash
sudo NAIVE_REPAIR=1 bash 3x-ui/setup.sh
```

This re-injects the Naive arm into `stream.conf` and reloads nginx + Caddy. Credentials, cert, Caddy binary, and Caddyfile are not touched. State (domain, panel/reality, backend port) comes from `/etc/naive-proxy/state.env`.

`/etc/nginx/conf.d/naive-acme.conf` and `/etc/cron.d/naive-cert-renew` survive on their own.

## Why webroot, not standalone

3x-ui Pro itself uses `certbot certonly --standalone` for the panel/reality certs, and stops nginx around the call. We avoid that here:

- **Webroot** lets renewals run while nginx stays up — no panel downtime.
- The webroot server in `conf.d/` survives 3x-ui Pro re-runs, so renewal keeps working without re-applying anything.
- It plays nicely with the per-cert `--cert-name` cron entry — only the Naive lineage is renewed by our cron, the deploy hook's `RENEWED_LINEAGE` check prevents Caddy reloads on unrelated renewals.

3x-ui Pro's own monthly cron (`certbot renew --nginx --post-hook "nginx -s reload"`) will also try to renew the Naive cert. That's redundant but not harmful — webroot validation works regardless of which cron picked it up first, and the deploy hook will fire either way.

## Service management

```bash
systemctl status caddy
systemctl reload caddy           # picks up Caddyfile changes
journalctl -u caddy -f
cat /etc/caddy/credentials.txt   # creds, URI
```

To change the cover site: edit `/etc/caddy/Caddyfile`, `systemctl reload caddy`. To rotate credentials: re-run the script — it'll regenerate them unless you keep the existing Caddyfile (the script offers credential reuse).

## Troubleshooting

**`certbot` fails with "challenge timed out".** DNS A-record for the Naive domain doesn't point at this server, OR `/etc/nginx/conf.d/naive-acme.conf` isn't being loaded. Check `nginx -T | grep -A5 naive-acme` and `dig +short A <naive_domain>`.

**`curl https://<naive_domain>` returns the panel fake-site, not Naive.** SNI mapping is wrong. Inspect `/etc/nginx/stream-enabled/stream.conf` — the Naive domain must have an `naive` arm before `default`. Re-run `NAIVE_REPAIR=1 bash 3x-ui/setup.sh`.

**Caddy logs `bad PROXY protocol header`.** nginx isn't sending PROXY v1 (or sending it with a different framing). Check `proxy_protocol on;` is present in stream `server` block. Should be — 3x-ui Pro writes it unconditionally.

**Caddy logs `permission denied` reading the cert.** Cert paths under `/etc/letsencrypt/live/<naive_domain>/` must be readable. The systemd unit runs as root, so this should be fine; if you've hardened the unit, check `ProtectSystem`/`PrivateMounts` settings.

**`nginx -t` fails after re-run of x-ui-pro.sh.** Expected — your stream patch was clobbered along with sites-enabled. Run `NAIVE_REPAIR=1`.

**Client gets HTTP 407 / Proxy auth required even with correct creds.** SagerNet-family clients need the URI exactly as printed (`naive+https://user:pass@host:443?padding=1#tag`). For Karing / sing-box-for-android, use the `singbox.json` block instead — its URI parser is unreliable.

## Sources

- 3x-ui Pro: <https://github.com/mozaroc/x-ui-pro>
- klzgrad/naiveproxy: <https://github.com/klzgrad/naiveproxy>
- caddy-forwardproxy (klzgrad fork): <https://github.com/klzgrad/forwardproxy/tree/naive>
- Caddy listener wrappers: <https://caddyserver.com/docs/caddyfile/options#listener-wrappers>
- nginx stream `ssl_preread`: <https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html>
