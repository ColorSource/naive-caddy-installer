# naive-caddy-installer

One-command NaiveProxy + Caddy installer for Debian/Ubuntu VPS.

## Features

- Builds Caddy with `klzgrad/forwardproxy@naive`
- Supports local static cover site or reverse-proxy cover site
- Automatically obtains a Let's Encrypt TLS certificate
- Generates NaiveProxy credentials, URI, QR code, CLI config, and sing-box outbound JSON
- Installs and manages `caddy.service`

## Requirements

- Debian 12 or Ubuntu 22.04+
- Public IPv4 VPS
- Domain A record pointing to the VPS
- Cloudflare proxy disabled / DNS-only
- Ports `80/tcp` and `443/tcp` open
- Root access

## Install

```bash
wget -qO- https://raw.githubusercontent.com/ColorSource/naive-caddy-installer/main/setup.sh | sudo bash
```

The installer asks for confirmation before making changes.

## Cover Modes

Unauthenticated visits to your domain are served by a normal cover site.

### Local static site

Default mode. Caddy serves files from:

```text
/var/www/naive-cover
```

If `index.html` does not exist, the installer creates a small placeholder page.

### Reverse proxy

Caddy reverse-proxies unauthenticated visits to an external cover site.

Example:

```text
https://www.lovense.com
```

## Non-interactive Examples

Static cover:

```bash
wget -qO- https://raw.githubusercontent.com/ColorSource/naive-caddy-installer/main/setup.sh | \
  sudo NAIVE_DOMAIN=proxy.example.com \
       NAIVE_COVER_MODE=static \
       NAIVE_STATIC_ROOT=/var/www/naive-cover \
       bash
```

Reverse-proxy cover:

```bash
wget -qO- https://raw.githubusercontent.com/ColorSource/naive-caddy-installer/main/setup.sh | \
  sudo NAIVE_DOMAIN=proxy.example.com \
       NAIVE_COVER_MODE=proxy \
       NAIVE_MASK_SITE=https://www.lovense.com \
       bash
```

## Output Files

| Path | Purpose |
|---|---|
| `/etc/caddy/Caddyfile` | Caddy + NaiveProxy config |
| `/etc/caddy/credentials.txt` | Credentials and URI |
| `/root/naive-client-config.json` | Naive CLI config |
| `/root/naive-singbox.json` | sing-box outbound JSON |
| `/var/www/naive-cover/index.html` | Static cover page, if static mode is used |

## Service Commands

```bash
systemctl status caddy
systemctl reload caddy
systemctl restart caddy
journalctl -u caddy -f
cat /etc/caddy/credentials.txt
```

## Notes

- Do not enable Cloudflare orange-cloud proxy for the NaiveProxy domain.
- Do not use sensitive paths as the static site root.
- This script does not configure firewall rules.
- This script does not enable BBR or tune kernel networking.

## Sources

- <https://github.com/klzgrad/naiveproxy>
- <https://github.com/klzgrad/forwardproxy/tree/naive>
- <https://caddyserver.com/>
