# Setup

Step-by-step install of Apollo on a fresh Linux server. Tested on Debian 13
(amd64) and Ubuntu 26.04 (arm64, Oracle Cloud). Every image is multi-arch,
so amd64 and arm64 hosts both work. Every command runs on the server as a
non-root user with `sudo`, from the repo root unless noted.

On a cloud VM, read [Provider notes](#provider-notes) first.

## 1. Prerequisites

Install these first, following each project's own instructions:

- [git](https://git-scm.com/downloads/linux) (minimal cloud images often
  lack it)
- [Docker Engine](https://docs.docker.com/engine/install/) from Docker's own
  apt repository, with the Compose plugin, and your user in the `docker`
  group ([post-install](https://docs.docker.com/engine/install/linux-postinstall/));
  log out and back in for the group to apply
- [Task](https://taskfile.dev/installation/)
- [Tailscale](https://tailscale.com/download/linux)
- [iptables-persistent](https://wiki.debian.org/iptables) (answer "No" when
  it offers to save the current rules). Some cloud images ship it
  preinstalled with their own rules; see [Provider notes](#provider-notes).

## 2. Join the tailnet

Portainer and NZBHydra2 are reachable only over Tailscale, and in step 8
SSH will be too.

```sh
sudo tailscale up
tailscale ip -4          # the server's Tailscale IP; you need it in step 4
```

From your workstation, check that `ssh <user>@<tailscale-ip>` works. Your
tailnet's access policy must let your workstation reach the server. Don't
move on until it works: the firewall in step 8 closes public SSH.

## 3. Clone and create the `.env` files

```sh
git clone --depth 1 https://github.com/feruzabad/apollo.git
cd apollo
for d in */; do [ -f "$d.env.example" ] && cp "$d.env.example" "$d.env"; done
chmod 600 */.env
```

## 4. Fill in each `.env`

Each `.env.example` documents its variables and how to generate each secret
(`openssl rand ...`). What to set:

| File               | Set                                                                                 |
|--------------------|-------------------------------------------------------------------------------------|
| `redis/.env`       | `PASSWORD`                                                                          |
| `caddy/.env`       | `AIOSTREAMS_DOMAIN`, `AIOMETADATA_DOMAIN`, `AIOMANAGER_DOMAIN`, `TLS` (see below)   |
| `aiostreams/.env`  | `BASE_URL`, `SECRET_KEY`, `AIOSTREAMS_AUTH`, `REDIS_PASSWORD`. The Hydra key comes in step 7 |
| `aiometadata/.env` | `HOST_NAME`, `ADMIN_KEY`, `ADDON_PASSWORD`, `REDIS_PASSWORD`; optionally the `BUILT_IN_*_API_KEY`s |
| `aiomanager/.env`  | `CORS_ORIGINS`, `ENCRYPTION_KEY`                                                    |
| `portainer/.env`   | `INTERFACE`: the Tailscale IP from step 2                                           |
| `nzbhydra2/.env`   | `INTERFACE`: the same Tailscale IP                                                  |
| `backup/.env`      | `RESTIC_PASSWORD`; optionally `RESTIC_REPOSITORY` + S3 credentials for offsite      |

These values must match across files:

- `redis/.env` `PASSWORD` = `REDIS_PASSWORD` in `aiostreams/.env` and
  `aiometadata/.env`
- `caddy/.env` `AIOSTREAMS_DOMAIN` ↔ `aiostreams/.env` `BASE_URL`
  (`https://<domain>`), and the same for `AIOMETADATA_DOMAIN` ↔ `HOST_NAME`
  and `AIOMANAGER_DOMAIN` ↔ `CORS_ORIGINS`

`SECRET_KEY` (AIOStreams) and `ENCRYPTION_KEY` (AIOManager) can't be changed
after the first start. Keep them, along with `RESTIC_PASSWORD`, in a password
manager: [RECOVERY.md](RECOVERY.md) needs `RESTIC_PASSWORD` to restore
everything else.

**TLS:**

- **Production:** create DNS A (and AAAA, if the server has IPv6) records for
  the three domains pointing at the server, and set `TLS=` (empty). Caddy
  fetches Let's Encrypt certificates on first start, so ports 80 and 443 must
  be reachable from the internet. On a cloud VM, that includes the
  provider's firewall (security group, security list, ...): allow 80/tcp,
  443/tcp and 443/udp in. With Cloudflare DNS, keep the records DNS-only
  (grey cloud), so Caddy can answer the ACME challenge itself.
- **Local/testing:** keep `TLS='tls internal'` (self-signed). Any hostnames
  work, e.g. `aiostreams.apollo.test`; reach them with a hosts-file entry or
  `curl -k --resolve <domain>:443:<server-ip> https://<domain>/`.

## 5. Make Docker wait for Tailscale at boot

Portainer and NZBHydra2 publish on the Tailscale IP. If Docker starts before
that IP exists, they fail and stay down.

```sh
sudo install -Dm644 host/wait-for-tailscale.conf /etc/systemd/system/docker.service.d/wait-for-tailscale.conf
sudo systemctl daemon-reload
```

## 6. Start the stack

```sh
task up
docker ps          # after a minute or two, everything is Up, and (healthy) where it has a check
```

`task up` creates the shared `apollo` and `apollo_edge` networks and starts
every service in order. They all restart with Docker, so the stack comes
back after a reboot on its own.

## 7. Post-install

1. **NZBHydra2 login.** Hydra starts without a login. Open
   `http://<tailscale-ip>:5076`, go to Config > Authorization, choose "Login
   form", add an admin user and restrict all sections. Leave Config >
   Downloading > NZB access type at "Proxy". Click **Save** at the top of
   the Config page: nothing is applied until you do. Check that it stuck:
   ```sh
   docker exec nzbhydra2 grep authType /config/nzbhydra.yml   # authType: "FORM"
   ```
2. **NZBHydra2 API key → AIOStreams.** Hydra generates the key on first
   start (Config > Main > API key). Copy it into `aiostreams/.env`
   `BUILTIN_NZBHYDRA_API_KEY`, then:
   ```sh
   task up:aiostreams
   ```
3. **Portainer.** Open `https://<tailscale-ip>:9443` (self-signed
   certificate) and create the admin user. The setup screen asks for a
   setup token, which Portainer prints on start:
   ```sh
   docker logs portainer 2>&1 | grep setup_token= | tail -1
   ```
   Portainer locks its setup if no admin is created within a few minutes.
   If that happens, restart it: `docker restart portainer`.
4. **AIOManager.** Open `https://<AIOMANAGER_DOMAIN>` and create your account.
   It's created on the first cloud sync. Then set
   `REGISTRATIONS_CLOSED=true` in `aiomanager/.env`, and run:
   ```sh
   task up:aiomanager
   ```
5. **AIOStreams / AIOMetadata.** Configure them at
   `https://<AIOSTREAMS_DOMAIN>` and `https://<AIOMETADATA_DOMAIN>`.
   Optionally put your AIOMetadata config UUID(s) in `aiometadata/.env`
   `CACHE_WARMUP_UUIDS` and run `task up:aiometadata`, to pre-warm catalogs
   daily.

## 8. Firewall

[`host/iptables/`](host/iptables) holds IPv4 and IPv6 rules for
iptables-persistent. Once they're in place:

- Public: only Caddy, on 80/tcp, 443/tcp and 443/udp (HTTP/3).
- Tailnet only: SSH, Portainer (9443), NZBHydra2 (5076). Everything else on
  the host is closed; Redis and the addons have no published ports at all.
- Outbound traffic stays open, since the addons need to reach debrid, usenet
  and metadata services.

Check again that SSH over Tailscale works (step 2). On Oracle Cloud, install
`rules.v4` as described in [Provider notes](#oracle-cloud) instead of with
the first line below. Then:

```sh
sudo install -m644 host/iptables/rules.v4 /etc/iptables/rules.v4
sudo install -m644 host/iptables/rules.v6 /etc/iptables/rules.v6
sudo sh -c 'iptables-restore < /etc/iptables/rules.v4 && ip6tables-restore < /etc/iptables/rules.v6 && systemctl restart tailscaled docker'
```

Keep these in mind:

- `iptables-restore` replaces the whole filter table, including the chains
  Docker and Tailscale add at runtime, so both are restarted in the same
  command. If your SSH session drops halfway, that still happens. At boot,
  the rules load before either one starts, so no action is needed then.
- Use `iptables-restore`, not `netfilter-persistent reload`. Whether
  `reload` flushes depends on `IPTABLES_RESTORE_NOFLUSH` in
  `/etc/default/netfilter-persistent`. Some cloud images set it, and then
  `reload` appends Apollo's rules behind the old ones, which keep deciding.
  Check with `sudo iptables -S INPUT`: it must start with `-P INPUT DROP`
  followed by Apollo's rules only.
- Never run `netfilter-persistent save`. It would write Docker's and
  Tailscale's runtime chains into the rule files. Edit `host/iptables/`
  and re-install instead.
- If you change `HTTP_PORT`/`HTTPS_PORT` in `caddy/.env`, change the ports in
  the `DOCKER-USER` rules to match.
- If you lock yourself out, use your provider's console to get in.

## 9. Verify

From a machine **outside** the tailnet:

```sh
nc -zv <public-ip> 443      # open
nc -zv <public-ip> 22       # times out
nc -zv <public-ip> 9443     # times out
curl -I https://<AIOSTREAMS_DOMAIN>/
```

From a machine **on** the tailnet:

```sh
ssh <user>@<tailscale-ip>
curl -kI https://<tailscale-ip>:9443/     # Portainer
curl -I http://<tailscale-ip>:5076/       # NZBHydra2
```

Finally, `sudo reboot`, and check that `docker ps` shows everything back up and
`sudo iptables -S INPUT` starts with `-P INPUT DROP`.

The backup container runs its first backup on start (`docker logs backup`),
then daily at 03:30 UTC.

## Provider notes

### Oracle Cloud

Tested on Oracle's Ubuntu 26.04 image (arm64).

- **Ingress rules.** The subnet's security list (or the instance's network
  security group) only allows SSH by default. Add ingress rules from
  `0.0.0.0/0` for 80/tcp, 443/tcp and 443/udp before step 6. The host
  firewall from step 8 still decides what gets through after that.
- **Preinstalled firewall.** The image ships iptables-persistent with its own
  `/etc/iptables/rules.v4`. Besides opening SSH, it has an `InstanceServices`
  chain that limits access to the instance's iSCSI boot volume and metadata
  services; each rule's comment points to Oracle's documentation on the
  security impact of removing it. Install Apollo's `rules.v4` with
  Oracle's chain appended, instead of the plain `install` in step 8:
  ```sh
  [ -f /etc/iptables/rules.v4.oracle ] || sudo cp /etc/iptables/rules.v4 /etc/iptables/rules.v4.oracle
  { sed '/^COMMIT/d' host/iptables/rules.v4
    echo ':InstanceServices - [0:0]'
    grep -E '^-A (OUTPUT|InstanceServices) ' /etc/iptables/rules.v4.oracle
    echo COMMIT
  } | sudo tee /etc/iptables/rules.v4 >/dev/null
  ```
  The image also sets `IPTABLES_RESTORE_NOFLUSH=yes`, which is why step 8
  loads the rules with `iptables-restore`.

## Troubleshooting

- **Image pulls fail with `lookup ... on 100.100.100.100:53: no such host`,
  but `getent hosts` resolves.** MagicDNS forwards to the server's own
  upstream resolver, and some resolvers (e.g. VMware's NAT DNS) break
  EDNS replies, which Docker's resolver needs. Check with
  `dig @100.100.100.100 registry-1.docker.io`. If the answer is malformed,
  point the server at working upstream resolvers (for dhcpcd:
  `static domain_name_servers=...` in `/etc/dhcpcd.conf`), then run
  `sudo systemctl restart tailscaled`.
- **Portainer or NZBHydra2 down after a reboot.** Docker started before
  Tailscale had its IP; check step 5. Run `task up:portainer` /
  `task up:nzbhydra2` to bring them back.
- **Caddy logs `failed to install root certificate ... read-only file
  system` with `tls internal`.** This is harmless. Caddy tries to add its
  local CA to the container's trust store, which is read-only; TLS still
  works.
