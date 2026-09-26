# Disaster Recovery

Restoring the full stack onto a fresh server after the original is lost.

This only works if the repo survived the server's loss: either
`RESTIC_REPOSITORY` was set to offsite storage (e.g. R2), or it was
local-only and you manually copied `backup/restic-repo/` somewhere else
yourself (another machine, external drive, etc.). A local-only repo that was
never copied off the server doesn't survive losing the server, since it's
on the same disk as everything else it backs up.

## Prerequisites on the new server

- Docker, Task, Tailscale installed and Tailscale authenticated (same as a
  normal install, see README).
- The bootstrap secrets, from a password manager (never only on the server
  itself): `RESTIC_PASSWORD`, and if using offsite storage also
  `RESTIC_REPOSITORY`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.
  Everything else comes back from the backup.

## Steps

1. Clone the repo:
   ```sh
   git clone <this-repo> apollo && cd apollo
   ```
   If restoring from a manually-copied local repo rather than offsite
   storage, also copy that directory to `apollo/backup/restic-repo/` now.

2. Pre-create the named volumes, empty. Don't start the app containers yet;
   restoring after they've initialized can conflict with the old data
   (`aiostreams/.env` `SECRET_KEY` in particular is tied to the restored
   configs).
   The labels mark them as Compose's own, as if `docker compose up` had
   created them; without them, Compose warns on every start that the volume
   "was not created by Docker Compose":
   ```sh
   for s in portainer caddy aiostreams aiometadata aiomanager prowlarr; do
     docker volume create \
       --label com.docker.compose.project=$s \
       --label com.docker.compose.volume=data \
       ${s}_data
   done
   ```

3. Restore everything in one shot, using a throwaway container with the same
   mount layout the backup job used, so `restic restore --target /` drops
   files back where they were read from.

   **Offsite repo (R2/S3-compatible):**
   ```sh
   export RESTIC_REPOSITORY=... RESTIC_PASSWORD=... \
          AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...

   docker run --rm \
     -e RESTIC_REPOSITORY -e RESTIC_PASSWORD -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY \
     -v portainer_data:/mnt/volumes/portainer \
     -v caddy_data:/mnt/volumes/caddy \
     -v aiostreams_data:/mnt/volumes/aiostreams \
     -v aiometadata_data:/mnt/volumes/aiometadata \
     -v aiomanager_data:/mnt/volumes/aiomanager \
     -v prowlarr_data:/mnt/volumes/prowlarr \
     -v "$(pwd)/restore-env:/mnt/volumes/env" \
     mazzolino/restic:1.8.2 restic restore latest --target /
   ```

   **Manually-copied local repo** (from step 1):
   ```sh
   export RESTIC_PASSWORD=...

   docker run --rm \
     -e RESTIC_REPOSITORY=/mnt/restic -e RESTIC_PASSWORD \
     -v "$(pwd)/backup/restic-repo:/mnt/restic:ro" \
     -v portainer_data:/mnt/volumes/portainer \
     -v caddy_data:/mnt/volumes/caddy \
     -v aiostreams_data:/mnt/volumes/aiostreams \
     -v aiometadata_data:/mnt/volumes/aiometadata \
     -v aiomanager_data:/mnt/volumes/aiomanager \
     -v prowlarr_data:/mnt/volumes/prowlarr \
     -v "$(pwd)/restore-env:/mnt/volumes/env" \
     mazzolino/restic:1.8.2 restic restore latest --target /
   ```

   Either way, this also drops every service's backed-up `.env` into
   `./restore-env/<service>/.env`.

4. Move the `.env` files into place:
   ```sh
   cp -R restore-env/. . && rm -rf restore-env
   ```

5. Update the values that are inherently tied to the old host:
   - `portainer/.env` `INTERFACE`: the new server's Tailscale IP
     (`tailscale ip -4`).
   - `prowlarr/.env` `INTERFACE`: the same Tailscale IP.

6. Bring the stack up:
   ```sh
   task up
   ```
   Caddy re-fetches Let's Encrypt certs for the external sites as needed,
   usually immediately since `caddy_data` (holding prior cert state) was
   restored.

7. Re-apply what lives outside the repo, since none of it was backed up:
   - Provider firewall rules (allow 443/tcp, 80/tcp, 443/udp, 41641/udp
     inbound, deny the rest).
   - DNS: if the server's public IP changed, update `AIOSTREAMS_DOMAIN` /
     `AIOMETADATA_DOMAIN` / `AIOMANAGER_DOMAIN`'s A/AAAA records.
   - Swap, on small-RAM hosts: a 2 GB `/swapfile` in `/etc/fstab` and
     `vm.swappiness=10` in `/etc/sysctl.d/`, so a memory spike slows the
     addons down instead of getting one OOM-killed.

8. Verify:
   ```sh
   docker compose ps
   tailscale status
   curl -I https://<AIOSTREAMS_DOMAIN>
   curl -I https://<AIOMETADATA_DOMAIN>
   curl -I https://<AIOMANAGER_DOMAIN>
   ```
   Confirm Portainer loads over the tailnet at `https://<tailscale-ip>:9443`,
   and Prowlarr at `http://<tailscale-ip>:9696`.
