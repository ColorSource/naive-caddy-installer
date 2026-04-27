# naive-caddy-installer

One-command installer for [NaïveProxy](https://github.com/klzgrad/naiveproxy) + [Caddy](https://caddyserver.com/) with `caddy-forwardproxy@naive` on Debian/Ubuntu VPS. The klzgrad stack: a single Caddy on :443 that serves both the Naive tunnel and a cover site.

## What it does

- Enables TCP BBR
- Installs the latest stable Go (resolved from go.dev)
- Builds Caddy with `klzgrad/forwardproxy@naive` via `xcaddy`
- Generates 16-char alphanumeric credentials
- Writes a `Caddyfile` with `basic_auth` + `probe_resistance` + `reverse_proxy` to a mask site
- Sets up `caddy.service` and waits for the Let's Encrypt TLS certificate
- Prints credentials, a CLI `config.json`, a `naive+https://` URI, a QR for it, and a sing-box outbound JSON block

## Run

```bash
wget -qO- https://raw.githubusercontent.com/Ieveltyanna/naive-caddy-installer/main/setup.sh | sudo bash
```

The script asks for the domain and the mask site; everything else (architecture, Go version, credentials) is detected.

### Non-interactive mode

```bash
wget -qO- https://raw.githubusercontent.com/Ieveltyanna/naive-caddy-installer/main/setup.sh | \
  sudo NAIVE_DOMAIN=proxy.example.com NAIVE_MASK_SITE=https://www.lovense.com bash
```

## Prerequisites

1. **VPS** running Debian 12 or Ubuntu 22.04+/24.04 with public IPv4. amd64 or arm64.
2. **Domain** with an A-record pointing at this VPS. Free subdomains (DuckDNS, etc.) work.
   - **No Cloudflare orange-cloud** — breaks both ACME and the Naive tunnel. Use grey-cloud / DNS-only, or a different DNS provider.
3. **Open :80 and :443** to the public internet. :80 is needed only at cert issuance (ACME HTTP-01); afterwards it can be closed (Caddy renews via TLS-ALPN).
4. **Root SSH access.**

## Picking a mask site

Caddy serves the chosen site to anyone hitting :443 without valid credentials. Closer TLS fingerprint to your IP-subnet neighbours = lower chance of mass scans flagging your VPS.

Run [RealiTLScanner](https://github.com/XTLS/RealiTLScanner) **from a different machine** (not the proxy VPS — it would log your IP in the same analytics bucket as the cover sites):

```bash
wget https://github.com/XTLS/RealiTLScanner/releases/latest/download/RealiTLScanner-linux-64
chmod +x RealiTLScanner-linux-64
./RealiTLScanner-linux-64 --addr <YOUR_VPS_IP>
```

Pick a `cert-domain` with `feasible=true`, `tls="TLS 1.3"`, `alpn=h2`, ideally `cert-issuer="Let's Encrypt"`. Pass it as `https://<domain>`.

If the scanner finds nothing useful — leave the default `https://www.lovense.com` (large, public, Let's Encrypt). **Don't use `https://demo.cloudreve.org`** — known fingerprint of mass-scanned naive servers.

## After installation

Files written:
- `/etc/caddy/credentials.txt` — login / password / URI (chmod 600)
- `/root/naive-client-config.json` — CLI client config (chmod 600)

Same content + a QR are also printed to stdout.

### CLI client (desktop)

1. Download `naive` for your OS: <https://github.com/klzgrad/naiveproxy/releases>
2. `scp root@<vps>:/root/naive-client-config.json .`
3. `./naive naive-client-config.json` — local SOCKS5 on `127.0.0.1:10808`.
4. Use as system / browser proxy or `ALL_PROXY=socks5h://127.0.0.1:10808`.

Verify:

```bash
curl --socks5-hostname 127.0.0.1:10808 https://ifconfig.me   # should return the VPS IP
curl -v https://your-domain.com                              # should serve the mask site
```

### Android

The de-facto URI across the SagerNet/NekoBox/NekoRay family:

```
naive+https://user:pass@host:port?padding=1#tag
```

`padding=1` enables HTTP/2 padding (the whole point of Naive); SagerNet-family clients respect it, others ignore it.

- **Exclave** (recommended): install [Exclave](https://github.com/dyhkwong/Exclave/releases) + the [Naïve plugin APK](https://github.com/klzgrad/naiveproxy/releases) → «+» → «Scan QR code» → Connect. URI parser: [`NaiveFmt.kt`](https://github.com/dyhkwong/Exclave/blob/dev/app/src/main/java/io/nekohasekai/sagernet/fmt/naive/NaiveFmt.kt#L28-L47).
- **NekoBox**: same flow as Exclave, same URI.
- **Karing / sing-box-for-android**: URI parsing is unreliable. Use the sing-box outbound JSON block the script prints — paste into `outbounds[]`.

## Service management

```bash
systemctl status caddy           # status
systemctl restart caddy          # restart
journalctl -u caddy -f           # live logs
cat /etc/caddy/credentials.txt   # creds (if you lost them)
```

Edit `/etc/caddy/Caddyfile` then `systemctl reload caddy` to change domain / mask / credentials. Or rerun the script — when an existing install is detected it offers a menu:

1. Rebuild Caddy from sources + regenerate Caddyfile (new credentials), restart
2. Keep existing Caddy binary + regenerate Caddyfile (new credentials), restart
3. **Reuse existing Caddyfile and credentials, just restart** — for pulling fresh QR / config without invalidating credentials you've already shared
4. Exit, leave system untouched

## Troubleshooting

**`acme: error: 403` / no cert.** Check A-record from the VPS itself (`dig +short your-domain.com`). Cloudflare must be grey-cloud. :80 must be reachable at issuance time.

**`address already in use`.** Something else listens on :443 — old Caddy, nginx, apache, container: `ss -tlnp | grep :443` and kill it. The script does this preflight, but a manual parallel Caddy will collide.

**`curl --socks5-hostname` returns client IP.** CLI `naive` isn't running or isn't listening on :10808: `ss -tlnp | grep 10808` on the client. Confirm `config.json` `user:pass` matches `/etc/caddy/credentials.txt`.

**`xcaddy build` fails with `no space left on device`.** Script uses `TMPDIR=/root/tmp`. Free space on `/root` or build elsewhere.

**Exclave: «Profile not found» / QR doesn't scan.** Copy the `naive+https://...` URI from the script's output, then «+» → «Add manual» → paste.

**Karing imports the URI but connection fails.** URI parser is unreliable across versions. Use the sing-box outbound JSON block instead.

## What this script does NOT do

- No firewall config (`ufw`, `firewalld`) — open :22, :80, :443/tcp manually
- No Cloudflare WARP for split ingress/egress IP
- No mobile client install — server-side only, plus URI/QR for client import
- No `apt upgrade` — risky on a loaded VPS without a snapshot
- No RHEL/Arch/Alpine support — Debian/Ubuntu only

## Sources

- Canonical klzgrad-stack gist: [swrneko/09e60de4d3d8f9a551a1a2c1ab9283c5](https://gist.github.com/swrneko/09e60de4d3d8f9a551a1a2c1ab9283c5)
- klzgrad/naiveproxy: <https://github.com/klzgrad/naiveproxy>
- caddy-forwardproxy (klzgrad fork): <https://github.com/klzgrad/forwardproxy/tree/naive>
- Exclave: <https://github.com/dyhkwong/Exclave>
- RealiTLScanner: <https://github.com/XTLS/RealiTLScanner>
