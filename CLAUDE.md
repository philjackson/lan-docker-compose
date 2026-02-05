# Docker Compose Home Setup

This repository contains docker compose configurations for a home server setup with Traefik as a reverse proxy.

**Note:** These files are accessed via network share. Docker commands cannot be run directly here - they must be run on the server itself.

## Directory Structure

```
/home/phil/pic/
├── [service-name]/
│   ├── compose.yml
│   └── .env (optional, for complex services)
├── dockcheck.sh (maintenance script)
└── regctl (utility binary)
```

Each service lives in its own directory with a single docker-compose file.

## Standard Environment Variables

Always include these in new services:

```yaml
environment:
  - PUID=1000
  - PGID=1000
  - TZ=Europe/London
```

## Restart Policy

All services use:

```yaml
restart: unless-stopped
```

## Networking

### Traefik Network

Most services connect to the external `traefik-net` network:

```yaml
networks:
  traefik-net:
    external: true
```

Services are then accessible at `http://[container-name].home.lan` (HTTP only, no HTTPS).

### Traefik Labels

For web-accessible services, add:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.services.[service-name].loadbalancer.server.port=[port]"
```

### Alternative Network Modes

- `network_mode: host` - For services needing direct host network access (e.g., syncthing)
- `network_mode: "service:gluetun"` - For VPN-tunneled services

## Volume Paths

**All service data must use bind mounts under `/home/pi/apps/[service-name]/`** - do not use named volumes.

| Purpose | Path |
|---------|------|
| Service data root | `/home/pi/apps/[service-name]/` |
| Config | `/home/pi/apps/[service-name]/config` |
| Database | `/home/pi/apps/[service-name]/postgres`, `/home/pi/apps/[service-name]/db`, etc. |
| Backups | `/home/pi/apps/[service-name]/backups` |
| Media library | `/home/pi/external/live/` (Film, Tv, Music subdirs) |
| Downloads | `/home/pi/external/downloads/` |

For multi-component services, use subdirectories:
```
/home/pi/apps/komodo/
├── postgres/
├── ferretdb/
├── backups/
└── periphery/
```

## Template for New Services

```yaml
services:
  servicename:
    image: org/image:latest
    container_name: servicename
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    volumes:
      - /home/pi/apps/servicename:/config
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.servicename.loadbalancer.server.port=8080"

networks:
  traefik-net:
    external: true
```

## Image Sources

Preferred registries:
- LinuxServer.io: `lscr.io/linuxserver/[service]:latest`
- GitHub Container Registry: `ghcr.io/[org]/[service]:[tag]`
- Docker Hub: `org/image:latest`

## Special Configurations

### Service Dependencies

For services requiring initialization order:

```yaml
depends_on:
  database:
    condition: service_healthy
```

### Hardware Access

For USB devices or special hardware:

```yaml
devices:
  - /dev/ttyUSB0:/dev/ttyUSB0
```

### Network Capabilities

For services needing network admin (DNS, VPN):

```yaml
cap_add:
  - NET_ADMIN
```

## Key Infrastructure Services

These services are referenced in the conventions above:

- **traefik** (ports 80, 443, 8082) - Reverse proxy. Provides the `traefik-net` network and `home.lan` routing.
- **gluetun** - VPN gateway. Services needing VPN use `network_mode: "service:gluetun"`.
- **pihole** (ports 53, 81) - DNS server. Requires `NET_ADMIN` capability.
