# Unbound (Docker)

A lightweight Docker image for [Unbound](https://nlnetlabs.nl/projects/unbound/about/), a validating, recursive, caching DNS resolver. Built on a minimal Debian/Ubuntu base with `apt-get` available and `dnsutils` included for easy troubleshooting. Works great as a **standalone resolver** or as **Pi-hole’s upstream**.

## Features
- Fresh base + packages at build time (nice with nightly GH Actions rebuilds)
- `unbound` + `dnsutils` (`dig`) baked in
- Healthcheck (queries `version.bind`)
- Small, readable config; runs fine without root privileges
- Ready for **GitHub Container Registry (GHCR)** + **Watchtower** auto-updates

---

## TL;DR (Quick start)

### 1) Minimal config files
Create two files in your repo under `unbound/`:

`unbound/unbound.conf`
```conf
server:
  # Listen on 5335 (common for Pi-hole upstream)
  interface: 0.0.0.0@5335
  interface: ::0@5335

  # Networking basics
  do-ip4: yes
  do-ip6: no
  do-udp: yes
  do-tcp: yes

  # Privacy & robustness
  hide-identity: yes
  hide-version: yes
  harden-glue: yes
  harden-dnssec-stripped: yes
  qname-minimisation: yes

  # Caching
  prefetch: yes

  # Root hints (optional but recommended if you ship a file)
  # Remove this line if you don't mount root.hints
  root-hints: "/etc/unbound/root.hints"

  # Access control (relax as needed for your network)
  access-control: 127.0.0.1/32 allow
  access-control: 172.16.0.0/12 allow
  access-control: 192.168.0.0/16 allow

# Uncomment to expose CHAOS TXT version for healthcheck (optional)
#server:
#  module-config: "validator iterator"
#  verbosity: 1
```

`unbound/root.hints` (optional): a copy of IANA root hints. You can update it occasionally; Unbound works without it, but supplying one avoids a startup fetch. (If you don’t want to ship this file, just remove the `root-hints:` line from the config.)

---

### 2) Run with Docker
```bash
docker run -d --name unbound \
  -p 5335:5335/tcp -p 5335:5335/udp \
  -v "$PWD/unbound/unbound.conf:/etc/unbound/unbound.conf:ro" \
  -v "$PWD/unbound/root.hints:/etc/unbound/root.hints:ro" \
  ghcr.io/<YOUR_GH_USERNAME>/unbound:latest
```

**Sanity check**
```bash
# Should print a version string or at least succeed
docker exec -it unbound dig @127.0.0.1 -p 5335 cloudflare.com A +short
```

---

## Docker Compose example
`docker-compose.yml`
```yaml
services:
  unbound:
    image: ghcr.io/<YOUR_GH_USERNAME>/unbound:latest
    container_name: unbound
    ports:
      - "5335:5335/tcp"
      - "5335:5335/udp"
    volumes:
      - ./unbound/unbound.conf:/etc/unbound/unbound.conf:ro
      - ./unbound/root.hints:/etc/unbound/root.hints:ro
    healthcheck:
      test: ["CMD-SHELL", "dig @127.0.0.1 -p 5335 version.bind TXT CHAOS +short || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped
    # Hardening (optional):
    # read_only: true
    # cap_drop: [ "ALL" ]
    # security_opt: [ "no-new-privileges:true" ]
```

Bring it up:
```bash
docker compose up -d
```

---

## Using with Pi-hole

If Pi-hole runs on the **same Docker network** as `unbound`, set Pi-hole’s upstream to the Unbound **service name** and port:

- Settings → DNS → Upstream DNS Servers → **Custom 1 (IPv4):** `http://unbound#5335`  
  (You can also use `unbound#5335` in newer Pi-hole builds.)

If Pi-hole runs elsewhere, point it at the host running Unbound, on port **5335**:
- Example: `192.168.1.10#5335`

Tip: keep `DNSSEC` enabled in Pi-hole; Unbound validates by default.

---

## Publish image to GHCR (auto-build & auto-update)

### 1) GitHub Actions workflow
Create `.github/workflows/build.yml`:
```yaml
name: Build & Push Unbound

on:
  push:
    branches: [ main ]
    paths:
      - "unbound/**"
      - ".github/workflows/build.yml"
  schedule:
    - cron: "0 8 * * *"  # daily at 08:00 UTC
  workflow_dispatch: {}

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build & Push
        uses: docker/build-push-action@v6
        with:
          context: ./unbound
          push: true
          pull: true        # always refresh base layers
          tags: |
            ghcr.io/${{ github.repository_owner }}/unbound:latest
            ghcr.io/${{ github.repository_owner }}/unbound:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

> This rebuilds nightly (and on changes), pulls the **latest Debian/Ubuntu base**, installs the latest `unbound` at build time, and pushes to `ghcr.io/<you>/unbound:latest`.

### 2) Switch Compose to the registry image
Make sure your compose references:
```yaml
image: ghcr.io/<YOUR_GH_USERNAME>/unbound:latest
```

### 3) Watchtower (optional, but handy)
If you use Watchtower for auto-updates, run it with registry auth and label-gating:

```yaml
watchtower:
  image: containrrr/watchtower:latest
  container_name: watchtower
  command: --label-enable --registry-auth --interval 3600 --cleanup
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
  restart: unless-stopped
```

Opt-in `unbound`:
```yaml
services:
  unbound:
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

> If your GHCR package is **private**, on the host do:  
> `docker login ghcr.io`  
> (Watchtower’s `--registry-auth` reuses those creds.)

---

## Building locally (no GHCR)
```bash
# Debian stable-slim (has apt-get)
docker build --pull -t unbound:local ./unbound
docker run -d --name unbound -p 5335:5335/udp -p 5335:5335/tcp \
  -v "$PWD/unbound/unbound.conf:/etc/unbound/unbound.conf:ro" \
  -v "$PWD/unbound/root.hints:/etc/unbound/root.hints:ro" unbound:local
```

Prefer Ubuntu? Swap the base in your `Dockerfile`:
```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    unbound dnsutils ca-certificates && rm -rf /var/lib/apt/lists/*
```

---

## Troubleshooting

- **Healthcheck failing / Pi-hole shows “no upstream”**  
  Check Unbound logs:
  ```bash
  docker logs unbound --tail=200
  ```
  Quick probe:
  ```bash
  docker exec -it unbound dig @127.0.0.1 -p 5335 example.com A +short
  ```

- **Windows/Docker Desktop line endings**  
  Ensure your config files are LF (not CRLF). In Git:
  ```bash
  git config core.autocrlf input
  ```
  Or convert once:
  ```bash
  dos2unix unbound/unbound.conf
  ```

- **Port conflicts**  
  Something else using 53/5335? Change mapped host ports in compose.

- **Root hints**  
  If you don’t mount `root.hints`, remove that line from `unbound.conf`. Unbound will still work by bootstrapping the root trust anchor.
