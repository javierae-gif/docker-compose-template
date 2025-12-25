# Docker Compose VPS Template - AI Agent Instructions

## Project Overview
This is a modular Docker Compose template for self-hosting 100+ applications on a VPS. The architecture uses:
- **Traefik v3** (reverse proxy, TLS termination via Let's Encrypt)
- **Authelia** (authentication & authorization layer)
- **Service Profiles** (enable/disable apps via `COMPOSE_PROFILES` env var)
- **Docker Compose Include Pattern** (root `compose.yaml` includes all `apps/*/compose.yaml`)

## Architecture & Critical Patterns

### Compose File Structure
- **Root compose.yaml** (`/opt/docker/compose.yaml`): Uses `include:` directive to merge 100+ app-specific compose files
- **Each app**: Located in `apps/{app-name}/` with `compose.yaml` + optional `.env` file
- **Profiles**: Every service has `profiles: [app-name, all]` for selective deployment
- **No root network definition**: Network is created at runtime from root `.env` `DOCKER_NETWORK` variable

### Key Infrastructure Services
1. **Traefik** (`apps/traefik/`): 
   - Listens on ports 80/443/853 (HTTP/HTTPS/DNS-over-TLS)
   - Expects services to expose via `traefik.http.routers.*` labels
   - All HTTPS redirects disabled (handled at entrypoint level)
   - Requires `DOCKER_NETWORK` env var (default: `aio_default`)

2. **Authelia** (`apps/authelia/`):
   - Forward-auth at `http://authelia:9091/api/authz/forward-auth`
   - Services apply via label: `traefik.http.middlewares.authelia@docker`
   - Requires template variables for environment substitution (`X_AUTHELIA_CONFIG_FILTERS: template`)
   - Special handling for Stremio addon domains (partial auth via `TEMPLATE_STREMIO_ADDON_HOSTNAMES`)

### Environment Configuration Hierarchy
1. **Root `.env`** (`/opt/docker/.env`): Core vars - `TZ`, `PUID`, `PGID`, `DOCKER_DIR`, `DOMAIN`, `LETSENCRYPT_EMAIL`, `COMPOSE_PROFILES`
2. **App `.env`** (e.g., `apps/gluetun/.env`): App-specific secrets & config (referenced via `env_file: - .env`)
3. **Hostname vars**: Central `.env` defines all subdomains via pattern `{APP}_HOSTNAME=${DOMAIN?}` (uses `${DOMAIN?}` for validation)

## Developer Workflows

### Deploy Services
```bash
# Start core infrastructure (Traefik + Authelia)
docker compose --profile required up -d

# Add services via COMPOSE_PROFILES
export COMPOSE_PROFILES="required,stremio-server,mediafusion"
docker compose up -d

# Or via --profile flag (overrides COMPOSE_PROFILES)
docker compose --profile prowlarr --profile jackett up -d
```

### Inspect & Debug
```bash
# View merged compose for a service
docker compose --profile {app} config | grep -A 50 "^services:"

# Check service health
docker compose logs -f {service-name}

# Verify variable substitution
docker compose config | grep -C 3 "{ENV_VAR}"
```

### Dependency Management
- Use `depends_on` with `condition: service_healthy` for ordered startup
- Services sharing resources (e.g., Zurg + rclone, Decypharr) must use matching mounts
- Mount point defaults: `${DOCKER_DATA_DIR}` = `/opt/docker/data/`, volumes in `/mnt/` are shared

## Common Implementation Patterns

### Service Template (Minimal)
```yaml
services:
  my-app:
    image: {image}
    container_name: my-app
    restart: unless-stopped
    expose: [8080]  # Internal only, no ports
    environment:
      - TZ=${TZ:-UTC}
      - PUID=${PUID}
      - PGID=${PGID}
    volumes:
      - ${DOCKER_DATA_DIR}/my-app:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-app.rule=Host(`${MY_APP_HOSTNAME?}`)"
      - "traefik.http.routers.my-app.entrypoints=websecure"
      - "traefik.http.routers.my-app.tls.certresolver=letsencrypt"
      - "traefik.http.routers.my-app.middlewares=authelia@docker"  # Add auth
    healthcheck:
      test: curl -f http://localhost:8080/health
      interval: 30s
      timeout: 10s
      retries: 3
    profiles:
      - my-app
      - all
```

### Environment Variable Validation
- Required vars: Use `${VAR?}` (fails if missing) e.g., `${DOMAIN?}`, `${PROWLARR_HOSTNAME?}`
- Optional vars: Use `${VAR:-default}` e.g., `${TZ:-UTC}`, `${DECYPHARR_CHANNEL:-latest}`
- Complex compositions: Allowable in hostname definitions, not recommended elsewhere

### Shared Dependencies Pattern
Apps may have helper services (e.g., mediafusion + mongodb/redis, decypharr + rclone):
- All services in a group share the same `profiles: [group-name, all]`
- Helper services set `depends_on: {main-service}: condition: service_healthy`
- Example: `mediafusion` (main) + `mediafusion_scheduler` + `browserless` + `mediafusion_mongodb` + `mediafusion_redis`

## Cross-Service Integration Points

### Volume Mounts (Critical)
- **Docker data**: `${DOCKER_DATA_DIR}/app-name:/config` (persistent state)
- **Host mounts**: `/mnt/remote/*` (for media, *arr services)
- **Config bind mounts**: `./config.json:/app/config.json` (from app folder)
- **Special**: Decypharr/Zurg use `/mnt:/mnt:rshared` for FUSE mounts

### Network Communication
- All services on single Docker network (configured via `DOCKER_NETWORK` env)
- Internal communication uses `service-name:port` (e.g., `authelia:9091`)
- Traefik label `traefik.http.routers.{service}.rule=Host(...)` controls external routing

### Config File Locations
- Traefik config: Not in compose (uses command-line flags)
- Authelia config: `apps/authelia/config/configuration.yml` (template-rendered)
- App configs: Typically `./config.json` or referenced via volumes

## Key Gotchas & Constraints

1. **Absolute paths only**: All `${DOCKER_*_DIR}` must be absolute paths; relative paths break when running from different directories
2. **Profile errors are silent**: If `COMPOSE_PROFILES` is unset or empty, no services start (except required)
3. **No bind mounts in subdirectories**: Ensure parent directories exist; Docker won't auto-create nested mounts
4. **Authelia template substitution**: Environment vars in `authelia/config/configuration.yml` only resolve if using `X_AUTHELIA_CONFIG_FILTERS: template`
5. **Hostname uniqueness**: All `*_HOSTNAME` values must be unique; Traefik routing conflicts if not
6. **Port conflicts**: Traefik reserves 80/443/853; apps must use `expose:` not `ports:` (except Traefik itself)

## When Adding New Services

1. Create `apps/new-app/compose.yaml` with service definition
2. Add service to root `include:` list
3. Define `{NEW_APP}_HOSTNAME` in root `.env` (pattern: `{app}.${DOMAIN}`)
4. Add Traefik labels (see template above)
5. Add Authelia middleware if authentication needed
6. Assign profile(s): at minimum `[new-app, all]`
7. Test: `docker compose --profile new-app config` then `docker compose --profile new-app up -d`

## Reference Files
- Root config: [.env](./.env), [compose.yaml](./compose.yaml)
- Traefik: [apps/traefik/compose.yaml](apps/traefik/compose.yaml) - reverse proxy config
- Authelia: [apps/authelia/compose.yaml](apps/authelia/compose.yaml), [config/configuration.yml](apps/authelia/config/configuration.yml)
- Complex app example: [apps/mediafusion/compose.yaml](apps/mediafusion/compose.yaml) - multi-service with dependencies
- Media server integration: [apps/decypharr/compose.yaml](apps/decypharr/compose.yaml) - FUSE mount example
