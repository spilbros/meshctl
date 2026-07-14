# Meshctl

Self-hosted [Headscale](https://github.com/juanfont/headscale) control server (open-source Tailscale-compatible coordination server), fully containerized behind Caddy with automatic TLS. Spin up your own mesh VPN coordination server in three commands.

```bash
git clone https://github.com/<your-username>/headscale-dockerized.git
cd headscale-dockerized
cp .env.example .env   # edit domain + base settings
docker compose up -d
```

That's it — Caddy issues a Let's Encrypt certificate automatically, headscale comes up behind it, and you're ready to register nodes.

## Why this exists

Tailscale's hosted coordination server is great, but it means trusting a third party with your device graph and DNS. Headscale is a self-hosted, open-source reimplementation of that same control plane — same client, same protocol, your own infrastructure. This repo packages it the way I wanted it for my own homelab: reproducible, documented, and copy-pasteable onto any VPS.

This is a companion project to [`gitea-dockerized`](#) — same philosophy: rootless/minimal containers, Caddy as the single TLS entrypoint, one `.env` to configure, no manual server-side setup.

## Architecture

```
Internet
   │
   │ HTTPS (443)
   ▼
┌─────────────┐
│    Caddy    │  ← automatic TLS via Let's Encrypt
└──────┬──────┘
       │ reverse_proxy (internal docker network)
       ▼
┌─────────────┐
│  headscale  │  ← control plane (gRPC + HTTP API)
└─────────────┘
       │
       ▼
   sqlite volume (node registry, keys, ACL state)
```

Headscale never exposes a port directly to the host — all traffic terminates TLS at Caddy and is proxied internally over the docker network.

## Stack

|Component|Role|
|---|---|
|`headscale/headscale`|Coordination server (Tailscale-compatible control plane)|
|`caddy:2-alpine`|Reverse proxy + automatic TLS termination|
|sqlite (volume)|Node registry, auth keys, ACL policy state|

## Getting started

### 1. Prerequisites

- A domain (or subdomain) pointing at your server's IP — **DNS-only / no CDN proxying** (Cloudflare orange-cloud breaks the gRPC traffic headscale needs)
- Docker + Docker Compose installed
- Port 443 open on the host

### 2. Configure

```bash
cp .env.example .env
```

Edit `.env`:

```env
MESH_DOMAIN=mesh.example.com   # your control-plane domain
MESH_HTTP_PORT=80              # host ports Caddy publishes
MESH_HTTPS_PORT=443
MESH_BASE_DOMAIN=home.arpa     # MagicDNS suffix for node hostnames
```

### 3. Launch

```bash
docker compose up -d
```

### 4. Create a user and register your first node

```bash
docker exec headscale headscale users create myuser
docker exec headscale headscale preauthkeys create --user myuser --expiration 1h
```

On the client device:

```bash
tailscale up --login-server=https://mesh.example.com --authkey=<key>
```

## Features

- [ ] One-command deploy via `docker compose up`
- [ ] Automatic TLS — zero manual certificate handling
- [ ] No exposed headscale port — Caddy is the only public surface
- [ ] **ACL / tagging scheme** — least-privilege access between nodes (e.g. personal devices vs. always-on server), deny-by-default policy in `acl.yaml`
- [ ] **Preauth key automation script** — one command to generate a key and print a ready-to-paste `tailscale up` command for onboarding new nodes
- [ ] Embedded DERP relay configuration (optional, for NAT traversal without relying on public Tailscale DERP servers)

## Project structure

```
headscale-dockerized/
├── docker-compose.yml
├── .env.example
├── headscale/
│   ├── config.yaml
│   └── acl.yaml
├── caddy/
│   └── Caddyfile
├── scripts/
│   └── new-node.sh        # preauth key + onboarding command generator
├── README.md
└── .gitignore
```

## Security notes

- Tagging and ACLs enforce deny-by-default access between nodes — an always-on internet-facing server doesn't get blanket trust just because it's on the same mesh as personal devices.
- Headscale's database (sqlite volume) holds node keys and registry state — back it up before any reinstall.
- No ports other than 443 need to be open on the host.

## License

MIT
