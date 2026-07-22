# Hermes Stack

Self-hosted [Hermes Workspace](https://github.com/outsourc-e/hermes-workspace) +
[Hermes Agent](https://github.com/NousResearch/hermes-agent), routed through
[9Router](https://hub.docker.com/r/decolua/9router) to Anthropic Claude, behind
Caddy with automatic HTTPS.

```
Internet ─▶ https://hermes.example.com ─▶ Caddy ─▶ Hermes Workspace
                                                     └─▶ Hermes Agent ─▶ 9Router ─▶ Anthropic Claude
```

Hermes never holds an Anthropic key. **9Router** stores the upstream provider
credentials; Hermes only ever talks to 9Router.

---

## Prerequisites

- A server (Linux) with a public IP.
- Two DNS records pointing at that server:
  - `hermes.example.com` → your server IP (required)
  - `router.example.com` → your server IP (only if you expose the 9Router dashboard)
- Ports **80** and **443** open to the internet.
- An Anthropic API key (added later, inside 9Router — not here).

## Install Docker

Docker Engine + Compose v2 plugin (Ubuntu/Debian):

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"   # log out/in afterwards
docker compose version            # confirm Compose v2 is present
```

## Clone

```bash
git clone <your-repo-url> hermes-stack
cd hermes-stack
```

## Configure

```bash
cp .env.example .env
```

Edit `.env` and set, at minimum:

| Variable | What to set |
|---|---|
| `HERMES_DOMAIN` | Your public hostname, e.g. `hermes.example.com` |
| `ACME_EMAIL` | Your email (Let's Encrypt) |
| `HERMES_PASSWORD` | A strong password for the Workspace login |
| `API_SERVER_KEY` | `openssl rand -hex 32` |
| `JWT_SECRET` | `openssl rand -hex 32` |
| `API_KEY_SECRET` | `openssl rand -hex 32` |
| `MACHINE_ID_SALT` | `openssl rand -hex 32` |
| `INITIAL_PASSWORD` | First-login password for the 9Router dashboard |
| `HERMES_UID` / `HERMES_GID` | Output of `id -u` / `id -g` |

Leave `ROUTER_API_KEY` **blank** for now — you create it after first boot
(see "Connecting Hermes to 9Router" below). Leave `ROUTER_DOMAIN` blank unless
you want the 9Router dashboard reachable on its own public hostname.

## Start

```bash
docker compose up -d
```

Caddy obtains certificates automatically on first run (allow a minute).
Check status and logs:

```bash
docker compose ps
docker compose logs -f
```

## Access URLs

| Service | URL |
|---|---|
| Hermes Workspace | `https://hermes.example.com` |
| 9Router dashboard | `https://router.example.com` *(only if `ROUTER_DOMAIN` is set)* |

If you did not set `ROUTER_DOMAIN`, reach the 9Router dashboard temporarily via
an SSH tunnel instead of exposing it:

```bash
ssh -L 20128:localhost:20128 user@your-server
docker compose exec 9router true   # (tunnel targets the published-nowhere port)
```

> The 9Router port is **not** published to the host by default. The simplest
> way to reach its dashboard is to set `ROUTER_DOMAIN`. Otherwise temporarily
> add a `ports: ["127.0.0.1:20128:20128"]` mapping to the `9router` service and
> tunnel to it.

## First login

**Hermes Workspace** — open `https://hermes.example.com`, log in with
`HERMES_PASSWORD`, and complete onboarding.

**9Router** — open its dashboard, log in with `INITIAL_PASSWORD`, and change
the password immediately.

## Adding your Anthropic API key inside 9Router

Hermes has no Anthropic key — 9Router does. In the 9Router dashboard:

1. Go to **Providers** and add **Anthropic**.
2. Paste your Anthropic API key.
3. Enable the Claude model(s) you want to route to.

9Router now talks to Anthropic on your behalf.

## Connecting Hermes to 9Router

1. In the 9Router dashboard, open **Settings → API Keys** and create a new key.
2. Copy it into `.env`:

   ```env
   ROUTER_API_KEY=<the key you just created>
   ```

3. Make sure `hermes/config.yaml` `model.default` matches a model/route you
   enabled in 9Router (edit it if needed).
4. Restart the agent so it picks up the key and config:

   ```bash
   docker compose up -d hermes-agent hermes-dashboard
   ```

Send a message in the Workspace — it now flows Workspace → Agent → 9Router →
Anthropic.

## Updating containers

```bash
docker compose pull        # fetch newer images
docker compose up -d       # recreate changed containers
docker image prune -f      # optional: reclaim disk
```

Persistent data (certificates, sessions, memory, 9Router DB) lives under
`./volumes/` and survives updates.
