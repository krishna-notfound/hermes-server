# Hermes Stack

Self-hosted Hermes Workspace + Hermes Agent, routed through 9Router, behind
Caddy with automatic HTTPS.

```text
Browser -> Caddy -> Hermes Workspace
Hermes Workspace -> Hermes Agent gateway -> 9Router -> model provider
```

Hermes Agent runs in the official Docker shape: one `nousresearch/hermes-agent`
container starts `gateway run`, and the dashboard is supervised inside that
same container with `HERMES_DASHBOARD=1`.

Do not run a separate `hermes-dashboard` container against the same
`./volumes/hermes` data directory. Current Hermes dashboard/gateway lifecycle,
profile detection, skills, and session APIs are designed to be co-located in
the supervised container.

## Prerequisites

- A Linux server with Docker Engine and Compose v2.
- DNS records pointing at the server:
  - `DASHBOARD_DOMAIN`, for the Hermes dashboard/workspace.
  - `ROUTER_DOMAIN`, for 9Router admin.
- Ports 80 and 443 open to the internet.

For a public EC2 deployment, protect both hostnames carefully. The included
Hermes dashboard basic-auth variables are the simplest self-hosted auth
provider, but the dashboard is still a sensitive admin surface.

## Configure

```bash
cp .env.example .env
```

Set at minimum:

| Variable | What to set |
|---|---|
| `DASHBOARD_DOMAIN` | Hermes dashboard/workspace hostname, for example `dashboard.example.com` |
| `ROUTER_DOMAIN` | 9Router hostname, for example `router.example.com` |
| `ACME_EMAIL` | Email for Let's Encrypt |
| `API_SERVER_KEY` | `openssl rand -hex 32` |
| `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` | Dashboard username |
| `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD` | Strong dashboard password |
| `HERMES_DASHBOARD_BASIC_AUTH_SECRET` | `openssl rand -hex 32` |
| `HERMES_PASSWORD` | Workspace login password |
| `JWT_SECRET` | `openssl rand -hex 32` |
| `API_KEY_SECRET` | `openssl rand -hex 32` |
| `MACHINE_ID_SALT` | `openssl rand -hex 32` |
| `INITIAL_PASSWORD` | First-login password for 9Router |
| `HERMES_UID` / `HERMES_GID` | Output of `id -u` / `id -g` |

Leave `ROUTER_API_KEY` blank for the first boot. Create it in 9Router, paste it
into `.env`, then restart Hermes.

## Start

```bash
docker compose up -d
docker compose ps
docker compose logs -f
```

## Access URLs

| Service | URL |
|---|---|
| Hermes dashboard/workspace | `https://${DASHBOARD_DOMAIN}` |
| 9Router dashboard | `https://${ROUTER_DOMAIN}` |

Log into the Hermes Agent dashboard with
`HERMES_DASHBOARD_BASIC_AUTH_USERNAME` and
`HERMES_DASHBOARD_BASIC_AUTH_PASSWORD`. This dashboard owns `/api/skills`,
`/api/sessions`, config, MCP, and admin APIs. The gateway on `:8642` owns the
OpenAI-compatible `/v1/*` API and health checks.

## Connecting Hermes to 9Router

1. Open the 9Router dashboard.
2. Add your upstream model provider credentials.
3. Create a 9Router API key.
4. Paste it into `.env` as `ROUTER_API_KEY`.
5. Verify `hermes/config.yaml` `model.default` matches a route/model enabled
   in 9Router.
6. Restart Hermes:

```bash
docker compose up -d hermes
```

## Updating

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

Persistent data lives under `./volumes/` and survives container updates.
