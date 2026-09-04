---
name: docker-compose-layered
description: Use when creating or extending a multi-service docker compose setup for any project — adding a new backing service (db, cache, broker, search, gateway), a new app service, or an observability component — or when asked to "set up docker compose", "add X to docker compose", or "organize the compose files".
---

# Layered Docker Compose

## Overview

Split compose config by **infrastructure layer** into separate files, joined
by one root file that only `include:`s them. Never put everything in one
giant `compose.yaml`, and never create a new top-level file for something
that already has a layer.

## When to Use

- Setting up docker compose for a new project from scratch.
- Adding a new backing service (db, cache, broker, search, gateway) to an existing compose setup.
- Adding a new app service or an observability component.
- Reviewing a compose setup for structure (one giant file vs. layered).

## Layer files

A root `compose.yaml` (or `docker-compose.yml`) with just:
```yaml
name: <project-name>
include:
  - compose.infrastructure.yaml   # stateful backing services: db, cache, broker, search, identity + their init containers
  - compose.services.yaml         # this project's own app/service containers (build: ...)
  - compose.gateway.yaml          # reverse proxy / API gateway, if any
  - compose.observability.yaml    # metrics/tracing/logs stack, if any
```
Only add a new layer file for a genuinely new *category* of infrastructure — a new backing service goes into the existing `compose.infrastructure.yaml`, not a new file.

Every layer file repeats:
```yaml
name: <project-name>
services:
  ...
networks:
  <project-network>:
    driver: bridge
```
(compose merges the `name` and `networks` blocks across included files)

## Per-service conventions

**Banner comments** group services by sub-layer, one banner per group, not per service:
```yaml
  # ═══════════════════════════════════════════════════════════════════════════
  # <LAYER NAME>
  # ═══════════════════════════════════════════════════════════════════════════
```

**Backing/third-party service:**
```yaml
  <service>:
    image: <image>:<pinned-tag>        # pin the tag — never :latest for anything with a healthcheck others depend on
    container_name: <service>
    restart: unless-stopped
    environment:
      <VAR>: ${<VAR>:-<dev-default>}   # every env var overridable, with a sane dev default
    ports:
      - "${<SERVICE>_PORT:-<default>}:<container-port>"
    volumes:
      - <service>-data:/path           # only if it has state worth persisting
    healthcheck:
      test: [...]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - <project-network>
```

**This project's own app service:**
```yaml
  <name>:
    build:
      context: <path-to-repo-root>
      dockerfile: <path-to-service>/Dockerfile
    restart: unless-stopped
    environment:
      # <Dependency 1, e.g. Postgres>
      <VAR>: ${<VAR>:-<default>}
      # <Dependency 2>
      ...
      # Observability
      OTEL_EXPORTER_OTLP_ENDPOINT: ${OTEL_EXPORTER_OTLP_ENDPOINT:-http://otel-collector:4318}
      ENVIRONMENT: ${ENVIRONMENT:-development}
    depends_on:
      <dep>:
        condition: service_healthy
    networks:
      - <project-network>
```
Group env vars with a `# <Dependency>` comment per block, one block per backing service it talks to, roughly in the order it depends on them, with an `# Observability` block last if applicable (what the app does with `OTEL_EXPORTER_OTLP_ENDPOINT` is covered in [[observability-logging]]).

**One-shot init/bootstrap container** (bucket/topic/index/schema setup after a backing service starts):
```yaml
  <service>-init:
    image: <minimal-image-with-a-shell>
    container_name: <service>-init
    restart: "no"
    depends_on:
      <service>:
        condition: service_healthy
    volumes:
      - ./docker/<service>/init.sh:/scripts/init.sh:ro
    entrypoint: ["/bin/sh", "/scripts/init.sh"]
    networks:
      - <project-network>
```
Put the script in a mounted file under a `docker/<service>/` directory next to the compose files — never inline a multi-line setup script in `command:`.

**Consumer of an init container** (an app service that needs the bootstrap done first) depends on it with `condition: service_completed_successfully` instead of `service_healthy`.

## Volumes

Named volumes only for services with persistent state, declared once at the bottom of the file that owns them. Skip a volume for stateless or init containers, and for caches that don't need to survive a restart.

## Wiring in a new service

1. Pick the layer file by category (table above), don't create a new file.
2. Add the service block following the per-service template that fits it.
3. Wire `depends_on` (with the right condition) on every service that talks to it, including the gateway if it's user-facing.
4. If the project has an observability stack, wire it in the same way as every other app service — don't invent a second observability path.
5. If the project has a task runner (Makefile, package.json scripts, etc.) with per-service run targets, add one for the new service, mirroring existing ones.

## Common Mistakes

- New top-level compose file instead of extending the matching layer file.
- Hardcoded values instead of `${VAR:-default}` — breaks per-environment overrides.
- Inline multi-line setup scripts in `command:` instead of a mounted `.sh` file.
- Missing `networks: [<project-network>]` on a new service — it can't reach the others.
- `:latest` on an image other services `depends_on` via healthcheck — pin it.
- One banner comment per service instead of per logical group.
