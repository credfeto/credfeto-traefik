# Documentation Instructions

[Back to Local Instructions Index](index.md)

## README Must Track Routing Changes

- Whenever a route, host, path or port is added to or changed in `dynamic_conf.template.yml` or `traefik.template.yml`, update `README.md` in the same change:
  - **P1.** The `## Services` entry: the host or path, the backend URL it forwards to, whether it is internal only (the `internal-only` middleware, not reachable through Cloudflare Tunnel) or a Cloudflare Tunnel origin, and any authentication it requires.
  - **P2.** Any subsection that explains the route's authentication.
  - **P3.** `## Port Mappings`, for every new entrypoint port.
- When a route, host, path or port is removed or renamed in either template, remove or rename it in `README.md` in the same change.
- `.env` values the templates substitute are config options under the global README rule in [../global/documentation.instructions.md](../global/documentation.instructions.md); in this repo its configuration section is the `.env` example block together with the `### Environment variables reference` table.
- Reason: `README.md` is the only place that lists what this Traefik instance exposes and where each route goes; the templates are not readable as an inventory, so a route missing from the README is effectively undocumented, and a stale entry misleads whoever deploys or debugs the host.
