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

## `.env.example` Must Track `.env` Changes

- Whenever a `.env` value is added, removed or renamed (a variable the templates substitute or one that `update` reads), update `.env.example` in the same change:
  - **P4.** Add each new key with a placeholder value, and a comment above it saying how to generate the real value (for example `openssl rand -hex 32`).
  - **P5.** Keep the keys in alphabetical order, because dotenv-linter's `UnorderedKey` check fails on any key out of order.
  - **P6.** Remove or rename keys to match, so `.env.example` never lists a key that is no longer used or misses one under its new name.
- Reason: `.env.example` is the template people copy to create `.env`, and `update` fails on any required key that is missing, so a key absent from `.env.example` breaks the first deployment of whoever follows it.
