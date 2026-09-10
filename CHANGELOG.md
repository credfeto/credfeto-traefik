# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

<!--
Please ADD ALL Changes to the UNRELEASED SECTION and not a specific release
-->

## [Unreleased]
### Security
- Restricted the monitoring direct port (457) to POST requests only, since VictoriaMetrics has no built-in authentication and the route was previously open to queries and admin operations from anyone able to reach it.
- Set a 2-year HSTS header (includeSubDomains, preload) on all HTTPS responses
### Added
- docs/ folder with architecture diagram showing repository folder structure
- cctv.markridgwell.com host routing to 192.168.60.180 over self-signed TLS
- keys.markridgwell.com zone pointing at 192.168.150.220:8081 using the wildcard certificate
- dns-05.markridgwell.com host routing to dual-stack backend 192.168.42.105 / 2a02:8010:61d5:42::105
- dns-06.markridgwell.com host routing to backend 192.168.42.106
- IPv6 support for the Docker network (NAT66 via the host's global address) so Traefik and backends can use IPv6
- IPv6 backend addresses (2a02:8010:61d5:42::101-106) restored for dns.markridgwell.com and dns-01..dns-06; IPv6 clients now route to the IPv6 backend via a ClientIP()-matched router, everything else stays on IPv4
- monitoring.markridgwell.com routing to 192.168.150.134:8428, with a dedicated Cloudflare Tunnel origin port (457) alongside the SNI/HTTPS route
- Add active health checks to the DNS routers - RFC 8484 DoH probe for the DNS-over-HTTPS service and the unauthenticated /api/status probe for the Technitium admin console services, so unhealthy DNS servers are automatically taken out of rotation
- Add health checks to photos, homeassistant, home, defi, linux-cache-01/02, dev-cache-01/02, github-api, cctv, keys and monitoring services (issue #37 Tier A/A'), each verified against the actual backend's own source or documented API behaviour
- Added Traefik failover services so IPv6-only DNS routers fall back to the IPv4 backend pool when all IPv6 backends are unhealthy, instead of returning an error
- Accept HTTP on port 80 and redirect it to HTTPS on all hosts
### Fixed
- YAML document start markers added to docker-compose.yml, dynamic_conf.template.yml, and traefik.yml to satisfy ansible-lint
- watchtower missing a restart policy, leaving it stopped after a Docker daemon restart
- api-nuget, funfair-nuget, funfair-prerelease-nuget and dotnet health checks were all silently probing the same default nginx vhost (builds.dotnet.local) instead of their own, since health checks bypass router middleware and the shared :5555 nginx has no default_server; added hostname: to each so they probe their own vhost, and added the missing npm health check the same way
- audiobookshelf-service pointed at port 8080, which is actually Dozzle on that host - Audiobookshelf itself is published on port 13378; also adds a /healthcheck health check now that the port is correct
- pacman-service and flathub-service health checks were silently probing the wrong :7777 nginx vhost (health checks bypass router middleware, and the shared :7777 backend has no default_server, so pacman's probe was hitting flathub.local's /ping instead of its own); added hostname: to each so they probe their own vhost, and corrected flathub's serversTransport serverName and flathub-header Host from flathub.markridgwell.com to flathub.local to match the backend's real vhost name
- traefik.service failed to start on a real Arch host: /etc/traefik was root:root with no group/other access, blocking the traefik user's own systemd unit from even reading traefik.yml, dynamic_conf.yml was root:root 600 so traefik couldn't read its routing config either, and ACME storage was pointed at a subdirectory (/etc/traefik/acme/acme.json) that the packaged unit's ProtectSystem=strict/ReadWritePaths sandbox doesn't allow writing to; storage moved to the flat /etc/traefik/acme.json path the unit actually allow-lists, and update now sets correct ownership/permissions on all three
- Every router relying on Let's Encrypt now explicitly declares the *.markridgwell.com wildcard as its certResolver domain, instead of leaving it unstated. Without an explicit domain, Traefik resolves each such router's certificate independently at startup, racing the wildcard's own issuance; on a cold cert store (no cached wildcard yet) this made every router request its own individual DNS-01 certificate in parallel, which failed outright for ollama.markridgwell.com since it has no public Cloudflare DNS zone (it's an internal-only host, never meant to get its own cert)
- traefik-api-token-middleware (backing photos-local-check) failed to load: the packaged unit's ProtectSystem=strict leaves no writable working directory for the plugin's own plugins-storage folder, which took the plugin down and made photos-direct-router invalid entirely. update now gives traefik.service a real writable WorkingDirectory=/var/lib/traefik, allow-listed via an added ReadWritePaths=
### Changed
- Re-addressed dns-01..dns-04 backends to 192.168.42.101-104 (was .251-.254) with new dual-stack IPv6 addresses 2a02:8010:61d5:42::101-104
- Replaced the Docker-based deployment (traefik, cloudflared, watchtower containers) with native Arch Linux systemd services: traefik and cloudflared are now installed from pacman, traefik.yml is generated from a new traefik.template.yml, and pacman hooks restart each service when its package is upgraded, replacing watchtower's role
### Deprecated
### Removed
- IPv6 backend addresses from dns-01..dns-05 services; backends are IPv4-only for now
### Deployment Changes

<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->

## [0.0.0] - Project created