# Deploying to a Raspberry Pi 5

This fork's `.github/workflows/build-pi5-image.yml` builds an arm64 Docker image on
every push to `feat-readarr` (or manually via "Run workflow") and publishes it to
GHCR at `ghcr.io/luca-bachini-barr/seerr-chaptarr` under tags `pi5-latest`,
`pi5-feat-readarr`, and `pi5-<short-sha>`.

This publishes a **container image**, not a flashable SD-card OS image — the app
already ships as a Docker container (see the repo's own `Dockerfile`/`compose.yaml`),
so the standard way to run it on a Pi 5 is via Docker, same as any other host.

## One-time setup on the Pi 5

1. Install Docker if you haven't already: `curl -fsSL https://get.docker.com | sh`
2. If the GHCR package is private, log in once: `docker login ghcr.io -u Luca-Bachini-Barr` (use a
   fine-grained PAT with `read:packages` as the password)
3. Create `docker-compose.yml`:

```yaml
services:
  seerr-chaptarr:
    image: ghcr.io/luca-bachini-barr/seerr-chaptarr:pi5-latest
    container_name: seerr-chaptarr
    restart: unless-stopped
    ports:
      - "5055:5055"
    volumes:
      - ./config:/app/config
    labels:
      - "com.centurylinklabs.watchtower.enable=true"

  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ${HOME}/.docker/config.json:/config.json:ro   # only needed if the GHCR package is private
    command: --interval 300 --label-enable --cleanup
```

4. `docker compose up -d`

## How the auto-update works

Watchtower polls every 5 minutes (`--interval 300`) for containers labeled
`com.centurylinklabs.watchtower.enable=true`, and only touches those
(`--label-enable`) so it won't also start auto-updating unrelated containers on the
same Pi. When the GitHub Actions workflow pushes a new `pi5-latest` image, Watchtower
pulls it, recreates the container, and removes the old image (`--cleanup`).

If you'd rather control exactly when updates land instead of fully automatic,
drop the `watchtower` service and instead re-run `docker compose pull && docker
compose up -d` manually (or via your own cron job) whenever you want to update.
