# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

<!--
Please ADD ALL Changes to the UNRELEASED SECTION and not a specific release
-->

## [Unreleased]
### Security
- Proxy egress allow-list (`firewall/configure`); logs only until `PROXY_FIREWALL_ENFORCE=true`
- DNS-over-TLS (`dot` TLS option) now requires TLS 1.3, matching the default for every other route
- Removed the Cloudflare Tunnel paths for `photos.markridgwell.com`; photos is now internal only
- Restricted monitoring port 457 to POST only, as VictoriaMetrics has no authentication
- Set a 2-year HSTS header (includeSubDomains, preload) on all HTTPS responses
- Applied the internal-only middleware to all internal routers so Cloudflare Tunnel is refused
### Added
- monitoring.markridgwell.com/mcp-funfair to the Funfair MCP server (8084), token checked in Traefik
- fail2ban `traefik-auth` jail: bans a client after 5 HTTP 401s in 10 minutes on any route
- logs.markridgwell.com to VictoriaLogs (9428) and /mcp to mcp-victorialogs (8083), both LAN only
- DNS-over-TLS for `dns.markridgwell.com` on port 853 (LAN only), proxied to Technitium over TCP
- docs/ folder with architecture diagram showing repository folder structure
- grafana.markridgwell.com to Grafana on the monitoring VM (192.168.150.134:3000), LAN only
- /mcp on grafana and monitoring to mcp-grafana (8082) and mcp-victoriametrics (8081), LAN only
- cctv.markridgwell.com host routing to 192.168.60.180 over self-signed TLS
- keys.markridgwell.com zone pointing at 192.168.150.220:8081 using the wildcard certificate
- dns-05.markridgwell.com host routing to dual-stack backend 192.168.42.105 / 2a02:8010:61d5:42::105
- dns-06.markridgwell.com host routing to backend 192.168.42.106
- IPv6 for the Docker network (NAT66 via the host's global address) so Traefik and backends use it
- IPv6 backends restored for dns and dns-01..dns-06, used by IPv6 clients via a ClientIP() router
- monitoring.markridgwell.com to 192.168.150.134:8428, plus Cloudflare Tunnel origin port 457
- Add DNS health checks (DoH probe, Technitium /api/status) so unhealthy servers leave rotation
- Health checks on photos, homeassistant, home, defi, caches, github-api, cctv, keys, monitoring
- Failover services let IPv6 DNS routers fall back to the IPv4 pool when all IPv6 backends fail
- Accept HTTP on port 80 and redirect it to HTTPS on all hosts
- familiar.markridgwell.com to 192.168.150.129:8081, LAN only via a new internal-only middleware
- Workflow MCP server at workflow-mcp.markridgwell.com, internal only (not via Cloudflare Tunnel)
- memory (/dev, /netadmin MCP) and wiki routes to 192.168.150.30; needs MEMORY_* .env values
- edge-familiar.markridgwell.com route to the Familiar webhook listener (port 8083), internal only
### Fixed
- `./update` re-runs itself after `git pull`, so update script changes apply on the same run
- DoT on 853 rejected Android Private DNS (ALPN `dot`); the router now uses a `dot` TLS option
- YAML document start markers added to the compose, dynamic and static configs for ansible-lint
- watchtower missing a restart policy, leaving it stopped after a Docker daemon restart
- NuGet and dotnet health checks hit the wrong vhost; now set hostname:, plus a new npm check
- audiobookshelf-service used port 8080 (Dozzle); now 13378 with a /healthcheck health check
- pacman/flathub checks hit the wrong vhost; added hostname: and use flathub.local for flathub
- traefik.service failed to start on Arch; fixed config permissions and moved ACME storage
- Routers declare the *.markridgwell.com wildcard, so none requests its own cert (ollama failed)
- Token plugin failed to load; traefik.service now has a writable /var/lib/traefik working dir
- README gave the wrong audiobookshelf port (8080, Dozzle); it now matches the proxy's 13378
- README now lists the cctv.markridgwell.com and keys.markridgwell.com routes, which were missing
- README "Adding a new service" used a non-existent web-secure entrypoint and no tls.domains
- ACME renewals failed via redirected port 53 and the cert expired; skip ANS checks, wait 10 min
- /mcp-funfair token renamed MCP_FUNFAIR_AUTH_TOKEN to match monitoring; update chmods .env to 600
### Changed
- Linux cache backends (pacman, flathub) moved to LinuxCaches VLAN 21: 192.168.210.200/201:7777
- Dev cache backends (NuGet, npm, dotnet) moved to Caches VLAN 20: 192.168.200.100/101:5555
- familiar.markridgwell.com moved to Familiar VLAN 19: 192.168.190.129:8081 (was 192.168.150.129)
- Access log keeps only 4xx/5xx or slow (>5s) requests, dropping ~16k unread lines an hour
- Monitoring VM backends moved to Monitoring VLAN 16: 192.168.160.134 (was 192.168.150.134)
- dns-01..dns-04 re-addressed to 192.168.42.101-104 and 2a02:8010:61d5:42::101-104 (was .251-.254)
- Docker deployment replaced by native Arch systemd services; pacman hooks replace watchtower
- Declare wildcard TLS domains once on the web-general entrypoint, not on every router
- LAN ranges defined once in update (IPv6 prefix from LAN_IPV6_PREFIX) and fed to internal-only
- Egress firewall now one atomic nftables table (`inet prx_egress`), replacing fragile firewalld
### Deprecated
### Removed
- dev-cache-01/02 and linux-cache-01/02 routes (they only reached Dozzle); logs go to VictoriaLogs
- Removed `home.markridgwell.com` entirely, including port 446: the homepage VM is retired
- IPv6 backend addresses from dns-01..dns-05 services; backends are IPv4-only for now
- No-op Host-rewrite middlewares that set Host to the value the router already matches
- Removed aur.markridgwell.com routing (routers, service, middleware, transport, port 451)
- Removed docker-registry.markridgwell.com routing as the registry is no longer running
### Deployment Changes
- Firewall rules also allow the LAN's global IPv6 /48 to ports 80 and 443; re-run ./update

<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->

## [0.0.0] - Project created