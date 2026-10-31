# Traefik Setup

Traefik v3 reverse proxy using Let's Encrypt TLS via Cloudflare DNS challenge.

## Resources

- [Changelog](CHANGELOG.md)
- [Contributing](CONTRIBUTING.md)
- [Security](SECURITY.md)

## Prerequisites

- Arch Linux host with `pacman`, `systemd`, and `firewalld`
- A Cloudflare account managing your domain's DNS
- A Cloudflare API token with **Zone → DNS → Edit** permission scoped to your domain

`traefik` and `cloudflared` are installed from Arch's official `extra` repository by `./update`
(`pacman -S --needed traefik cloudflared`); no manual package installation is required first.
Ongoing package updates are handled by whatever mechanism keeps this host's packages current
(e.g. an existing ansible-pull setup) - `./update` installs a pacman hook for each service so it
restarts automatically when its package is upgraded.

## First-time setup

### 1. Create the `.env` file

Copy the example and fill in your credentials:

```bash
cp .env.example .env
```

Edit `.env`:

```env
CF_DNS_API_TOKEN=your_cloudflare_api_token
CLOUDFLARE_EMAIL=your@email.com
CLOUDFLARE_TUNNEL_TOKEN=your_cloudflare_tunnel_token
# Shared agent memory (credfeto-memory-docker): one bearer token per space
# (openssl rand -hex 32), and the wiki's htpasswd line
# (htpasswd -nbB mark '<password>'; single $ here, this file is not compose).
MEMORY_DEV_TOKEN=your_memory_dev_token
MEMORY_NETADMIN_TOKEN=your_memory_netadmin_token
MEMORY_WIKI_USERS=mark:$2y$05$replace_me
PHOTOS_LOCAL_SECRET=your_photos_x_local_secret
```

`.env` is gitignored and will not be committed.

### Environment variables reference

| Variable | Required | Used by | Purpose |
| -------- | -------- | ------- | ------- |
| `CLOUDFLARE_EMAIL` | Yes | `traefik` | Templated into `traefik.yml`'s `certificatesResolvers.letsencrypt.acme.email` for Let's Encrypt registration. |
| `CF_DNS_API_TOKEN` | Yes | `traefik` | Cloudflare DNS API token used by the ACME DNS challenge provider; read live from `.env` via `traefik.service`'s `EnvironmentFile=`. |
| `CLOUDFLARE_TUNNEL_TOKEN` | Yes | `cloudflared` | Consumed once by `cloudflared service install` when `./update` first bootstraps the tunnel service; not re-read afterwards. |
| `PHOTOS_LOCAL_SECRET` | Yes | `./update` | Shared secret injected into the photos router rule. Requests to port 450 must include `X-Local: <value>` matching this secret; unauthorised requests are rejected. Configure Cloudflare Tunnel to add this header on all requests to the photos origin. |
| `MEMORY_DEV_TOKEN` | Yes | `./update` | Bearer token for the `memory.markridgwell.com/dev` space. Requests must include `Authorization: Bearer <value>`. Generate with `openssl rand -hex 32`. |
| `MEMORY_NETADMIN_TOKEN` | Yes | `./update` | Bearer token for the `memory.markridgwell.com/netadmin` space. Requests must include `Authorization: Bearer <value>`. Generate with `openssl rand -hex 32`. |
| `MEMORY_WIKI_USERS` | Yes | `./update` | Basic auth user for `wiki.markridgwell.com`. One `htpasswd -nbB mark '<password>'` line, with single `$` characters and no quotes; holds one user only. |

### 2. Start Traefik

```bash
./update
```

The `update` script validates `.env`, installs/updates the `traefik` and `cloudflared` packages,
installs pacman hooks that restart each service on its own package upgrade, generates
`traefik.yml` and `dynamic_conf.yml` from their templates, sets up ACME storage and firewall
rules, bootstraps the cloudflared tunnel service on first run, and enables/starts both services.

## Configuration files

| File | Purpose |
| ---- | ------- |
| `traefik.template.yml` | Static configuration template: entrypoints, ACME, providers; committed to git |
| `dynamic_conf.template.yml` | Dynamic configuration template: routers, services, middlewares; committed to git |
| `/etc/traefik/traefik.yml` | Generated from `traefik.template.yml` by `./update`; deployed on the host, never committed |
| `/etc/traefik/dynamic_conf.yml` | Generated from `dynamic_conf.template.yml` by `./update`; deployed on the host, never committed |
| `/etc/traefik/acme.json` | Let's Encrypt certificate storage |

## Adding a new service

In `dynamic_conf.template.yml`, add a router and service following the existing pattern. Traefik forwards the original `Host` header by default, so add a header middleware only when the backend expects a different `Host` (as `my-service.local` does here):

```yaml
http:
  routers:
    my-router:
      rule: "Host(`my-service.markridgwell.com`)"
      service: my-service
      entryPoints:
        - web-general
      middlewares:
        - my-header

  services:
    my-service:
      loadBalancer:
        servers:
          - url: "https://192.168.10.10:5555"
          - url: "https://192.168.10.20:5555"

  middlewares:
    my-header:
      headers:
        customRequestHeaders:
          Host: "my-service.local"
```

The `markridgwell.com` / `*.markridgwell.com` wildcard certificate is declared once, under `entryPoints.web-general.http.tls` in `traefik.template.yml`, so every router on the `web-general` entrypoint shares it. New `web-general` routers therefore need no `tls` block. For a service that should only be reachable from the LAN, also add the existing `internal-only` middleware to the router's `middlewares` list.

The `internal-only` middleware is an `ipAllowList` that admits the private IPv4 ranges, IPv6 unique local and link-local addresses, and the LAN's global IPv6 prefix. It checks the TCP remote address of the connection, not `X-Forwarded-For`. Loopback is deliberately excluded: `cloudflared` connects to Traefik over loopback, so excluding it means requests arriving through Cloudflare Tunnel are refused with `403 Forbidden` on these hosts.

Traefik watches `dynamic_conf.yml` for changes and reloads automatically; no restart required for route or service changes. To apply template changes that alter `${PHOTOS_LOCAL_SECRET}` or other substituted values, re-run `./update`.

## DNS services

The following DNS routes are configured in `dynamic_conf.template.yml`. Every `dns*.markridgwell.com` router, both the `-v6` and the plain router, carries the `internal-only` middleware, so these hosts are not reachable through Cloudflare Tunnel:

| Hostname | IPv4 upstream target | IPv6 upstream target |
| -------- | --------------------- | --------------------- |
| `dns.markridgwell.com` | `https://192.168.42.101`, `.102`, `.103`, `.104`, `.105`, `.106` | `https://[2a02:8010:61d5:42::101]`, `::102`, `::103`, `::104`, `::105`, `::106` |
| `dns-01.markridgwell.com` | `https://192.168.42.101:53443` | `https://[2a02:8010:61d5:42::101]:53443` |
| `dns-02.markridgwell.com` | `https://192.168.42.102:53443` | `https://[2a02:8010:61d5:42::102]:53443` |
| `dns-03.markridgwell.com` | `https://192.168.42.103:53443` | `https://[2a02:8010:61d5:42::103]:53443` |
| `dns-04.markridgwell.com` | `https://192.168.42.104:53443` | `https://[2a02:8010:61d5:42::104]:53443` |
| `dns-05.markridgwell.com` | `https://192.168.42.105:53443` | `https://[2a02:8010:61d5:42::105]:53443` |
| `dns-06.markridgwell.com` | `https://192.168.42.106:53443` | `https://[2a02:8010:61d5:42::106]:53443` |

### TLS and Host behaviour for `dns.markridgwell.com`

- Upstream TLS uses a dedicated `serversTransports.dns` transport.
- `insecureSkipVerify: true` is enabled to allow self-signed backend certificates.
- `serverName: "dns.markridgwell.com"` is set so SNI matches the backend certificate name.
- The original `Host: dns.markridgwell.com` header is forwarded unchanged, because Traefik's default `passHostHeader: true` applies.

The `dns-01` to `dns-06` routes forward to HTTPS backends on port `53443` via the shared `serversTransports.dns-admin` transport, and forward the original `Host` header unchanged.

### IPv6-aware routing

Each `dns*.markridgwell.com` host has two routers: a `-v6` router matching `ClientIP(`::/0`)` at `priority: 100`, and the plain `Host()` router at `priority: 1` as the fallback. A request arriving over IPv6 matches the `-v6` router and is forwarded to the `-v6` service (the IPv6 backend addresses above); everything else (IPv4 clients, or any host without a `-v6` router defined) falls through to the plain router and IPv4 backend. This requires the client's real IP to reach Traefik unmodified; it relies on `forwardedHeaders.trustedIPs` in `traefik.yml` being scoped to private ranges only, so a public client's `ClientIP()` reflects their actual connection rather than a spoofable header.

## Dashboard

The Traefik dashboard is available at `https://proxy.markridgwell.com` once running (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel).

## Services

All services are accessible via HTTPS on port 443 with Let's Encrypt certificates unless stated otherwise.

### Development Cache Services

- `dev-cache-01.markridgwell.com` → `http://192.168.150.100:8080` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `dev-cache-02.markridgwell.com` → `http://192.168.150.101:8080` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `linux-cache-01.markridgwell.com` → `http://192.168.150.200:8080` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `linux-cache-02.markridgwell.com` → `http://192.168.150.201:8080` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)

### Docker Services

- `docker-cache.markridgwell.com` → `http://192.168.150.202:8081` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)

### Git / Workflow Services

- `github-api.markridgwell.com` → `http://192.168.150.15:3000` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `git-workflow.markridgwell.com` → `https://192.168.150.15:8081` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `workflow-mcp.markridgwell.com` → `https://192.168.150.15:8091` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)

### Key Management

- `keys.markridgwell.com` → `https://192.168.150.220:8081` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)

### Monitoring

- `monitoring.markridgwell.com` → `http://192.168.150.134:8428`, also available on port **457** (HTTP direct, POST only; see below)

#### Monitoring: write-only direct port (457)

Port 457 is the Cloudflare Tunnel origin for `monitoring.markridgwell.com`, intended for remote Telegraf agents to push metrics into VictoriaMetrics, which has no built-in authentication. The router on port 457 only matches `POST` requests (`` Method(`POST`) ``); any other method (queries, deletes, admin calls) does not match the route and is rejected. The main `:443` route is unrestricted, for querying/dashboard access.

### Home Automation

- `home.markridgwell.com` → `http://192.168.150.150:8080`, also available on port **446** (HTTP direct)
- `homeassistant.markridgwell.com` → `http://192.168.34.8:8123` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `audiobookshelf.markridgwell.com` → `http://192.168.150.16:13378` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `familiar.markridgwell.com` → `http://192.168.150.129:8081` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `defi.markridgwell.com` → `https://192.168.150.12`, also available on port **447** (HTTP direct)
- `cctv.markridgwell.com` → `https://192.168.60.180` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)

### Media

- `photos.markridgwell.com` → `http://192.168.150.154:2283`, also available on port **450** (HTTP direct, X-Local header required)

#### Photos: X-Local header authentication (port 450)

Port 450 is the Cloudflare Tunnel origin for `photos.markridgwell.com`. To prevent direct access that bypasses Cloudflare, Traefik validates a shared-secret header on all requests arriving on this port:

- Requests with `X-Local: <PHOTOS_LOCAL_SECRET>` are forwarded to the Immich backend.
- All other requests are rejected by the `traefik-api-token-middleware` plugin.

**Configuration:** Set `PHOTOS_LOCAL_SECRET` in `.env`, then configure the Cloudflare Tunnel origin rule for `photos.markridgwell.com` to add the request header `X-Local: <same value>`. The Cloudflare dashboard path is: **Zero Trust → Networks → Tunnels → [your tunnel] → Public Hostnames → photos.markridgwell.com → Additional application settings → HTTP Settings → Request headers**.

### NuGet Registries

- `api-nuget.markridgwell.com` → `https://192.168.150.100:5555`, `https://192.168.150.101:5555`, also available on port **449** (HTTP direct)
- `funfair-nuget.markridgwell.com` → `https://192.168.150.100:5555`, `https://192.168.150.101:5555`, also available on port **455** (HTTP direct)
- `funfair-prerelease-nuget.markridgwell.com` → `https://192.168.150.100:5555`, `https://192.168.150.101:5555`, also available on port **456** (HTTP direct)

### NPM Registry

- `npm.markridgwell.com` → `https://192.168.150.100:5555`, `https://192.168.150.101:5555`, also available on port **448** (HTTP direct)

### .NET Build Artefacts

- `dotnet.markridgwell.com` → `https://192.168.150.100:5555`, `https://192.168.150.101:5555` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)

### AI Services

- `ollama.markridgwell.com` → `https://192.168.150.10:5555`, `https://192.168.150.20:5555` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `memory.markridgwell.com/dev` → `http://192.168.150.30:8101` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel; bearer token required)
- `memory.markridgwell.com/netadmin` → `http://192.168.150.30:8102` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel; bearer token required)
- `wiki.markridgwell.com` → `http://192.168.150.30:8103` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel; basic auth required)

#### AI Memory: per-space bearer tokens and wiki basic auth

Each memory space on `memory.markridgwell.com` has its own path prefix and its own bearer token:

- Traefik strips the path prefix (`/dev` or `/netadmin`) before forwarding the request to the space's backend.
- The `traefik-api-token-middleware` plugin checks `Authorization: Bearer <token>` against that space's token (`MEMORY_DEV_TOKEN` or `MEMORY_NETADMIN_TOKEN`). It rejects requests without a matching token, and removes the header before forwarding when the token matches.

`wiki.markridgwell.com` uses Traefik basic auth with the single user in `MEMORY_WIKI_USERS`.

### Linux Package Mirrors

- `aur-repo.markridgwell.com` → `https://192.168.150.10:5555`, `https://192.168.150.20:5555` (internal only via the `internal-only` middleware; not reachable through Cloudflare Tunnel)
- `pacman.markridgwell.com` → `https://192.168.150.200:7777`, `https://192.168.150.201:7777`, also available on port **452** (HTTP direct)
- `flathub.markridgwell.com` → `https://192.168.150.200:7777`, `https://192.168.150.201:7777`, also available on port **453** (HTTP direct)

## Port Mappings

The following table lists all exposed ports and their purposes:

- **80** (TCP): HTTP entrypoint; redirects every request to the 443 entrypoint over HTTPS
- **443** (TCP/UDP): HTTPS entrypoint; all services with Let's Encrypt TLS + HTTP/3

All HTTPS responses on port 443 carry a `Strict-Transport-Security` header
(`max-age=63072000; includeSubDomains; preload`, i.e. 2 years).

The ports below are plain HTTP used as Cloudflare Tunnel origin services.
**Each hostname must have its own dedicated port**: Cloudflare Tunnel ingress rules
route by hostname, and each rule maps to a single origin URL (including port).

- **446** (TCP): Home Service, HTTP (Cloudflare Tunnel origin)
- **447** (TCP): DeFi Service, HTTP (Cloudflare Tunnel origin)
- **448** (TCP): NPM Registry, HTTP (Cloudflare Tunnel origin)
- **449** (TCP): API NuGet Registry, HTTP (Cloudflare Tunnel origin)
- **450** (TCP): Photos service, HTTP (Cloudflare Tunnel origin)
- **452** (TCP): Pacman Cache, HTTP (Cloudflare Tunnel origin)
- **453** (TCP): Flathub Repository, HTTP (Cloudflare Tunnel origin)
- **455** (TCP): FunFair NuGet Registry, HTTP (Cloudflare Tunnel origin)
- **456** (TCP): FunFair Pre-release NuGet Registry, HTTP (Cloudflare Tunnel origin)
- **457** (TCP): Monitoring, HTTP (Cloudflare Tunnel origin)

TLS is terminated by Cloudflare on the public side. Traefik does not re-encrypt
the internal leg from `cloudflared` to itself.
