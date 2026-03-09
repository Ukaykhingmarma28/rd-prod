# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Production Docker Compose stack for a WordPress + WooCommerce site. Runs behind Cloudflare (handles SSL termination) with Traefik as the ingress router on the host.

## Architecture

```
Traefik (host) → Nginx (reverse proxy + FastCGI cache) → WordPress (PHP-FPM) → MariaDB
                                                        → Redis (object cache)
```

All services run on a Docker overlay network (`wordpress_net`). WordPress content is stored in the `wordpress_data` named volume, shared read-write with the WordPress container and read-only with Nginx.

### Services & Resource Limits

| Service   | Image                      | CPU | RAM    | Notes                              |
|-----------|----------------------------|-----|--------|------------------------------------|
| mariadb   | mariadb:11                 | 2   | 2048M  | InnoDB buffer pool = 1200M         |
| redis     | redis:7-alpine             | 0.5 | 512M   | allkeys-lru, 512MB maxmemory      |
| wordpress | wordpress:6-fpm-alpine     | 1   | 2048M  | PHP-FPM dynamic, up to 40 workers  |
| nginx     | nginx:1.25-alpine          | 0.5 | 512M   | FastCGI cache 1GB disk, 256MB keys |

## Key Configuration Files

- `docker-compose.yml` — Service definitions, volumes, networks, resource limits, Traefik labels
- `.env.example` — Required environment variables (copy to `.env` and fill in)
- `nginx/nginx.conf` — Worker tuning, gzip, FastCGI cache zone definition
- `nginx/default.conf` — Server block: cache bypass rules (logged-in users, WooCommerce cart/checkout, POST, wp-admin), security headers, static asset caching, Cloudflare HTTPS detection via `CF-Visitor`
- `php/custom.ini` — PHP limits (512M memory, 128M uploads) and OPcache settings (`validate_timestamps=0` — requires cache flush on code deploy)
- `php/www.conf` — PHP-FPM pool: dynamic PM, 40 max children, slow log at 5s, status/ping endpoints
- `mariadb/custom.cnf` — InnoDB tuning, slow query log (>1s), utf8mb4

## Common Commands

```bash
# Start the stack
docker compose up -d

# View logs
docker compose logs -f [service]

# Restart a single service
docker compose restart nginx

# Enter a container shell
docker compose exec wordpress sh
docker compose exec mariadb mysql -u root -p

# Flush FastCGI cache
docker compose exec nginx rm -rf /var/cache/nginx/fastcgi/*

# Check PHP-FPM status (from within the network)
docker compose exec nginx curl http://wordpress:9000/status
```

## Important Design Decisions

- **SSL is handled by Cloudflare**, not this stack. Nginx listens on port 80. The `CF-Visitor` header is mapped to pass `HTTPS=on` to PHP so WordPress generates correct URLs.
- **FastCGI cache bypasses** are critical for WooCommerce: logged-in users, cart, checkout, my-account, POST requests, and wp-admin all skip the cache. The `X-Cache-Status` response header shows HIT/MISS/BYPASS.
- **OPcache `validate_timestamps=0`** means PHP never checks if files changed on disk. After deploying code/plugin updates, restart the WordPress container or flush OPcache.
- **Traefik integration** is via deploy labels on the nginx service. The `Host` rule uses the `DOMAIN` env var.
- The network uses `overlay` driver (for Docker Swarm compatibility). For single-host Docker Compose, this works with `attachable: true`.
