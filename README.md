# Apollo

> A self-hosted streaming stack for Stremio/Nuvio: addons, metadata, and the
> infrastructure to run them, all behind a single reverse proxy.

![Docker Compose](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)
![Caddy](https://img.shields.io/badge/reverse%20proxy-caddy-1F88C0?logo=caddy&logoColor=white)
![Self Hosted](https://img.shields.io/badge/self--hosted-yes-success)
![License](https://img.shields.io/badge/license-MIT-yellow)

Apollo bundles a [Stremio](https://www.stremio.com/)/[Nuvio](https://github.com/NuvioMedia)
streaming addon ([AIOStreams](https://github.com/Viren070/AIOStreams)) and a
metadata addon ([AIOMetadata](https://github.com/cedya77/aiometadata)) and an
account/addon manager ([AIOManager](https://github.com/Sonicx161/AIOManager)) behind
[Caddy](https://caddyserver.com/), with [NZBHydra2](https://github.com/theotherp/nzbhydra2)
feeding AIOStreams usenet indexers, [Portainer](https://www.portainer.io/) for
container management, [Redis](https://redis.io/) for caching, and automated
[restic](https://restic.net/) backups, orchestrated end-to-end with [Task](https://taskfile.dev).

This is an opinionated setup, not a general-purpose template: **Tailscale is
mandatory**, not an optional extra. Portainer and NZBHydra2 have no public
exposure path; their web UIs (Portainer on 9443 with its own self-signed HTTPS,
NZBHydra2 on 5076) are bound only to the host's Tailscale IP. There's no
fallback for running this without a tailnet.

## Features

- AIOStreams: unified streaming addon for Stremio/Nuvio clients
- AIOMetadata: metadata addon, with its built-in image cache and an on-disk
  metadata tier (cold store) behind Redis, each on a separate volume that
  backups skip, since both can be rebuilt; catalogs can be pre-warmed daily
- AIOManager: addon management and sync across Stremio/Nuvio accounts,
  with registrations closed once your own account exists
- Caddy in front of the public sites, with real Let's Encrypt certs
- Portainer for container management, reachable only over the tailnet at
  `https://<tailscale-ip>:9443`
- NZBHydra2 for usenet indexers: it downloads NZBs itself and hands them to
  AIOStreams, so indexers only ever see Hydra. Web UI only over the tailnet at
  `http://<tailscale-ip>:5076`
- Automated restic backup/prune/check jobs, covering every service's `.env`
  alongside its data volume; local by default, optionally offsite (e.g.
  Cloudflare R2)
- One `compose.yaml` per service, sharing an external Docker network
- Pinned image versions everywhere, no floating `latest` tags
- Hardened containers: all capabilities dropped (only what each image needs
  is added back), `no-new-privileges`, PID limits, read-only root filesystem
  for Caddy and Redis
- `task up` / `task down` for the whole stack or a single service

## Architecture

```
                         ┌──────────────┐
   Internet ─────────────▶    Caddy     │──────▶ AIOStreams (external)
                         │ (reverse     │──────▶ AIOMetadata (external)
                         │  proxy)      │──────▶ AIOManager (external)
                         └──────────────┘
                                │
                         apollo network
                                │
                    ┌───────────────────────┐
                    │  Redis  │  Backups     │
                    └───────────────────────┘

   Tailscale ────────────▶ Portainer :9443 (internal only)
   Tailscale ────────────▶ NZBHydra2 :5076 (internal only, AIOStreams via apollo)
```

Every service lives in its own folder with its own `compose.yaml` and
`.env`, all attached to one external `apollo` Docker network. There's no
monolithic compose file; spin services up and down independently, or all
together.

Caddy additionally joins `apollo_edge`, an IPv6-enabled network that carries
its published ports. Docker then forwards IPv6 clients by kernel NAT instead
of through `docker-proxy`, so Caddy (and the addons behind it) see real client
addresses on IPv6 too. The addons stay IPv4-only, which keeps their outbound
traffic to debrid/usenet services on a single address.

## Services

| Service       | Folder         | Description                          | Exposure                    |
|---------------|----------------|---------------------------------------|------------------------------|
| `portainer`   | `portainer/`   | Docker management UI                  | Internal (Tailscale) only    |
| `caddy`       | `caddy/`       | Reverse proxy, automatic HTTPS         | External                     |
| `aiostreams`  | `aiostreams/`  | Stremio/Nuvio streaming addon          | External                     |
| `aiometadata` | `aiometadata/` | Stremio/Nuvio metadata addon           | External                     |
| `aiomanager`  | `aiomanager/`  | Stremio/Nuvio account/addon manager    | External                     |
| `nzbhydra2`   | `nzbhydra2/`   | Usenet indexer manager for AIOStreams   | Internal (Tailscale) only    |
| `redis`       | `redis/`       | Cache shared by AIOStreams/AIOMetadata | Internal (apollo network)    |
| `backup`      | `backup/`      | restic backup / prune / check jobs     | n/a                           |

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/installation/)
- [Tailscale](https://tailscale.com/), installed and running on the host:
  required, not optional. Portainer and NZBHydra2 are only reachable over it.

### Installation

```sh
git clone <this-repo>
cd apollo
for d in */; do [ -f "$d.env.example" ] && cp "$d.env.example" "$d.env"; done
# fill in each */.env with your domains, IPs, and secrets
sudo install -Dm644 host/wait-for-tailscale.conf /etc/systemd/system/docker.service.d/wait-for-tailscale.conf
sudo systemctl daemon-reload
task up
```

Task creates the shared networks and brings every service up in the
right order. Every service restarts with Docker (`restart: unless-stopped`),
so the stack comes back on boot by itself. The `host/` drop-in makes Docker
wait for Tailscale first: services publishing ports on the Tailscale IP
would fail to start, and stay down, if Docker won the race.

### Usage

```sh
task up                  # start everything
task down                # stop everything
task up:aiostreams        # start a single service
task down:caddy           # stop a single service
task --list                # see all available tasks
```

## Configuration

Each service reads only its own `<service>/.env` (see the `.env.example`
next to it for the full, documented template), so every `compose.yaml` runs
on its own. Key things you'll want to set:

- `portainer/.env` `INTERFACE`: the host's Tailscale IP
  (`tailscale ip -4`); Portainer binds here only
- `nzbhydra2/.env` `INTERFACE`: the same Tailscale IP; Hydra's UI binds here
  only. Hydra starts without a login, so right after the first `task up` set
  Config > Authorization to Login form, add an admin user and restrict all
  sections. Keep Config > Downloading > NZB access type at Proxy
- `caddy/.env` `AIOSTREAMS_DOMAIN` / `AIOMETADATA_DOMAIN` /
  `AIOMANAGER_DOMAIN`: public hostnames for each service
- `caddy/.env` `TLS`: `tls internal` for local dev, empty in production
  (real Let's Encrypt certs, issued automatically)
- `backup/.env` `RESTIC_PASSWORD`: encrypts your backup repository
- `backup/.env` `RESTIC_REPOSITORY`: optional restic backend URL for offsite
  backups (e.g. Cloudflare R2); leave empty for local-only

A few values describe a connection between two services, so they must be
set to the same value in both files:

- Redis password: `redis/.env` `PASSWORD`, and `REDIS_PASSWORD` in
  `aiostreams/.env` and `aiometadata/.env`
- NZBHydra2 API key: generated by Hydra (Config > Main), copied into
  `aiostreams/.env` `BUILTIN_NZBHYDRA_API_KEY`
- Public URLs: `caddy/.env` domains and each app's own base URL

`.env` files are git-ignored, never commit them. The `.env.example` files
are the tracked, secret-free templates.

See [RECOVERY.md](RECOVERY.md) for restoring the stack onto a fresh server
from backup.

## License

[MIT](LICENSE)

## Acknowledgements

- [AIOStreams](https://github.com/Viren070/AIOStreams)
- [AIOMetadata](https://github.com/cedya77/aiometadata)
- [AIOManager](https://github.com/Sonicx161/AIOManager)
- [Caddy](https://github.com/caddyserver/caddy)
- [NZBHydra2](https://github.com/theotherp/nzbhydra2)
- [Portainer](https://github.com/portainer/portainer)
