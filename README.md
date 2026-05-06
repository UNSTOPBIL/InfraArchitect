example

# Project README

This repository contains resources and documentation for [project name]. This README has been updated to document a new small attack/defense lab topology added to the repository test documentation (see test.md). The new lab provides a deliberately vulnerable web application backed by MongoDB and a Kali host for attacker/testing purposes.

> WARNING: The lab intentionally includes insecure components. See the Security & Isolation section below before running anything.

---

## Table of Contents

- Overview & Purpose
- Architecture
- Services
- Prerequisites
- Quickstart (example docker-compose)
- Application stack (docker-compose) — recent change
- Docker Compose: Recommended Best Practices
- Ports & Accessibility
- Persistence & Data Management
- Security & Isolation Warnings
- Cleanup & Resource Management
- Troubleshooting
- How to view the docker-compose diff locally
- Credits, License & Responsible Use
- Changelog

---

## Overview & Purpose

The repository was extended to document a small local lab topology intended for experimentation, training, and testing security tooling. The components introduced and documented in test.md are:

- Kali Linux — attacker / analysis host (interactive)
- Vulnerable Web Application — intentionally insecure target application (HTTP)
- MongoDB — backend data store for the vulnerable application

Intended use cases:
- Local training and hands-on exercises
- Validation of defensive tooling, scanners, and monitoring
- Reproducible testing of attacks against a known vulnerable app

This README provides guidance and example configuration to help safely deploy the topology locally. The repository currently contains documentation (test.md). Example automation (docker-compose.yml) is provided below as a suggested starting point — adapt to your environment.

What changed in this update
- Added test.md describing the small attack/defense lab topology (Kali attacker, Vulnerable Web App, MongoDB backend).
- Added an example docker-compose.yml (documented in this README) to help get the lab up quickly.
- Documented recommended network isolation, persistence, ports, and security warnings.
- Added guidance and suggested best practices for docker-compose usage (pinning tags, named volumes, healthchecks, secrets, .env usage).
- NOTE (recent compose change): The repository's default docker-compose configuration has been changed from a monitoring/observability stack to an application stack. See "Application stack (docker-compose) — recent change" below for details and implications.

---

## Architecture

Simple logical layout:

Attacker (Kali)
  |
  | (internal testing network)
  |
Vulnerable Web App <---> MongoDB (internal DB)

- The web app communicates with MongoDB on the internal network.
- The web app may be exposed to the host for testing (e.g., host:8080 -> container:80).
- MongoDB should be internal-only (no host/public mapping) unless you intentionally require external DB access.

ASCII diagram:

```
[ Host / Dev Machine ]
    |
    +--[kali] (attacker VM/container)  -- optional host ports (ssh/gui)
    |
    +--[vuln-web] (target)  <-->  [mongodb] (internal DB)
             | host:8080 -> container:80
             | (mongodb:27017 only on internal network)
```

---

## Services

- Kali Linux (attacker)
  - Purpose: provide attacker tools for testing and analysis.
  - Notes: may require interactive/privileged container or a VM image. Choose appropriate image/tag (kali-rolling or official VM).
  - Ports: expose only what you need (SSH/GUI optional). Prefer not to expose to public networks.

- Vulnerable Web Application (target)
  - Purpose: intentionally insecure application for testing.
  - Common ports: 80 (HTTP) / 443 (HTTPS)
  - Notes: may require environment variables, seed data, or build steps. The repository documents the app behavior in test.md.

- MongoDB (backend)
  - Purpose: data store for the vulnerable app.
  - Default port: 27017
  - Important: verify whether authentication is enabled. By default in this lab it may be lax; treat as untrusted. Use a named volume for persistence.

- NOTE: The repository's docker-compose.yml was recently changed to launch a different application stack by default (see "Application stack (docker-compose) — recent change"). That compose now brings up frontend-vcl, product-catalogue, order-service, db (MongoDB), and cache (Redis) instead of the previous monitoring/logging services (Prometheus/Loki/Grafana/etc.). If you expect monitoring services to be available via compose, update or add a monitoring compose file.

---

## Prerequisites

Minimum suggestions:
- Docker (20.x+ recommended)
- Docker Compose CLI (docker compose) or docker-compose; the example support references Compose file version 3.9
- CPU: 2+ cores (more for Kali GUI)
- Memory: 4GB+ (increase for Kali and multiple services)
- Disk: space for images and MongoDB volume
- Internet access to pull images (unless images are provided locally)

Optional:
- Virtual machine host (e.g., VirtualBox/VMware) if you prefer full VM isolation for Kali.

Credentials & data:
- If the vulnerable app or MongoDB require credentials or seed data, consult test.md or supply environment variables when starting.

---

## Quickstart (example docker-compose)

The repository contains documentation (test.md), but may not include runnable IaC. Below is a recommended example docker-compose.yml you can adapt and save as `docker-compose.yml`. This example demonstrates an isolated bridge network, internal-only MongoDB, and a host port mapping for the web app:

```yaml
version: "3.8"
services:
  kali:
    image: kalilinux/kali-rolling
    container_name: lab_kali
    tty: true
    stdin_open: true
    networks:
      - labnet
    # Optional: expose ports only if needed (SSH, VNC, etc.)
    # ports:
    #   - "2222:22"

  vuln-web:
    image: your-vuln-app-image:latest
    container_name: lab_vuln_web
    environment:
      - MONGO_URI=mongodb://mongodb:27017/vulndb
    networks:
      - labnet
    ports:
      - "8080:80"  # host port -> container port (adjust as needed)
    depends_on:
      - mongodb

  mongodb:
    image: mongo:6
    container_name: lab_mongodb
    networks:
      - labnet
    volumes:
      - mongo_data:/data/db
    # Do NOT publish MongoDB port to the host unless you explicitly need to.
    # ports:
    #   - "27017:27017"  # avoid enabling this in production or on public networks

networks:
  labnet:
    driver: bridge

volumes:
  mongo_data:
```

Notes:
- Replace `your-vuln-app-image:latest` with the actual vulnerable-app image or build steps.
- The `kali` service above is configured as a container. You may choose to use a full VM for better isolation.
- Do not publish MongoDB to the host unless required. If you do expose it, enable authentication and firewall rules.

Sample commands:
- Start: docker-compose up -d (or docker compose up -d)
- View logs: docker-compose logs -f vuln-web
- Stop and remove containers (preserve volumes): docker-compose down
- Stop and remove containers + volumes: docker-compose down --volumes --remove-orphans

---

## Application stack (docker-compose) — recent change

Summary of change
- The repository's default compose was updated: version was bumped from 3.8 -> 3.9 and the composition changed from a monitoring/observability stack to an application stack.
- Removed monitoring services (deleted): prometheus, node-exporter, loki, promtail, grafana, postgres-db.
- Added application services: frontend-vcl (nginx:alpine), product-catalogue (node:18-alpine), order-service (python:3.11-slim), db (mongo:6.0), cache (redis:7.0-alpine).
- Exposed ports and mounts changed:
  - Monitoring ports (9090, 9100, 3100, 3000) were removed.
  - New frontend exposes 80:80 on the host.
  - MongoDB data is persisted to a named volume prod_data.

Why this matters
- Functional shift: Running the repository's default docker-compose up will now bring up an application stack (frontend, two app services, MongoDB, Redis) rather than Prometheus/Loki/Grafana/etc. If you previously relied on the monitoring stack coming up from compose, it will no longer be present.
- Persistence: MongoDB data will persist in a named volume prod_data.
- CI/automation: Pipelines or scripts that expected the monitoring stack or Postgres will break until updated.
- Security: Postgres password from the old compose was removed. The new MongoDB instance is started without authentication (typical for local dev but not for production).

Example minimal snippet (reflects the new compose layout — adapt before use):

```yaml
version: "3.9"
services:
  frontend-vcl:
    image: nginx:alpine
    ports:
      - "80:80"
    depends_on:
      - product-catalogue
      - order-service

  product-catalogue:
    image: node:18-alpine
    environment:
      - MONGO_URL=mongodb://db:27017/products
    depends_on:
      - db

  order-service:
    image: python:3.11-slim
    environment:
      - REDIS_HOST=cache
    depends_on:
      - cache

  db:
    image: mongo:6.0
    volumes:
      - prod_data:/data/db

  cache:
    image: redis:7.0-alpine

volumes:
  prod_data:
```

Important technical notes and potential issues
- Base images only (no build contexts): product-catalogue and order-service currently reference plain base images (node:18-alpine and python:3.11-slim). There are no build: contexts, no volumes mounting local source directories, and no command entries to start application code. In this state:
  - These containers will start a base image and likely exit or sit idle (they will not run your app unless you add a command or mount the code).
  - You probably intended to either add build contexts (Dockerfiles) or mount source code and a start command.
- depends_on does not guarantee readiness: it only controls startup order. Without healthchecks, product-catalogue or order-service may attempt to connect to db/cache before they are fully ready.
- No healthchecks or restart policies are present — consider adding healthcheck and restart: unless-stopped or restart: always for local dev resilience.
- No secrets/credentials: MongoDB is started without authentication. This is acceptable for local dev but must not be used in production.
- Hardcoded envs: MONGO_URL and REDIS_HOST are present in compose; prefer using an .env file or secret management for overrides.

Developer guidance (how to get the application services running locally)
- Option A — Add Dockerfiles and use build contexts (recommended for production-like builds):
  - Create Dockerfile in each service folder and update compose:
    product-catalogue:
      build: ./product-catalogue
    order-service:
      build: ./order-service
  - Ensure the Dockerfiles copy source code, install deps, and specify CMD.

- Option B — Mount source code for local development and add a start command:
  - Example (product-catalogue):
    product-catalogue:
      image: node:18-alpine
      volumes:
        - ./product-catalogue:/usr/src/app
      working_dir: /usr/src/app
      command: ["npm", "start"]
  - Example (order-service):
    order-service:
      image: python:3.11-slim
      volumes:
        - ./order-service:/app
      working_dir: /app
      command: ["python", "-m", "uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]

- Healthchecks / readiness:
  - Add healthcheck blocks for db/cache and app services and use wait-for scripts or health-based orchestration to ensure readiness before app starts connecting.

- Example .env (create at repo root and reference variables in compose):
  ```
  COMPOSE_FILE_VERSION=3.9
  FRONTEND_IMAGE=nginx:alpine
  PRODUCT_IMAGE=node:18-alpine
  ORDER_IMAGE=python:3.11-slim
  MONGO_IMAGE=mongo:6.0
  REDIS_IMAGE=redis:7.0-alpine
  PROD_DATA_VOLUME=prod_data
  FRONTEND_PORT=80
  MONGO_URL=mongodb://db:27017/products
  REDIS_HOST=cache
  ```

Commands to run
- Start (detached): docker compose up -d
- Start and rebuild (after adding Dockerfiles): docker compose up --build -d
- View logs: docker compose logs -f frontend-vcl
- Stop and remove containers (preserve volumes): docker compose down
- Stop and remove containers + volumes: docker compose down --volumes --remove-orphans
- Remove named volume manually: docker volume rm repo_prod_data (volume name will include project prefix)

Observability impact
- The previous monitoring stack (Prometheus, Loki, Grafana, node-exporter) was removed from the default compose. If you require local observability, consider:
  - Re-adding a separate docker-compose.monitoring.yml with the monitoring services.
  - Using service-based metrics exporters and a local Prometheus scraping config.
  - Keeping monitoring and application compose files separate for modularity.

CI / automation impact
- Any CI jobs or scripts that relied on the previous monitoring services or Postgres will need updates to either:
  - Adjust to the new services (Mongo/Redis), or
  - Use a separate compose file for monitoring to continue starting those services.

Suggested immediate improvements to the repo (optional)
- Add Dockerfiles for product-catalogue and order-service and switch compose to use build contexts.
- Mount source code for local development (volumes) and specify commands (npm start, gunicorn/uvicorn, etc.).
- Add healthchecks and restart policies (restart: unless-stopped) for better reliability.
- Externalize configuration into .env and provide an example .env.template.
- Add a separate monitoring compose (docker-compose.monitoring.yml) if observability is still desired.
- Add documentation describing how to seed the database and run integration tests locally.

---

## Docker Compose: Recommended Best Practices

Based on the recent documentation additions and common docker-compose considerations, consider adopting these practices in your compose files:

- Pin image tags for reproducibility
  - Avoid `:latest` in production/testing CI. Use explicit tags (e.g., `mongo:6.0.6`, `your-vuln-app:1.0.0`).
  - Document image versions used for training exercises.

- Use named volumes for persistence
  - Named volumes (as in `mongo_data` or `prod_data`) keep data across container recreation.
  - Document how to back up/export/import volumes if students need persistent seeded data.

- Keep databases internal-only
  - Do not publish DB ports unless strictly necessary. If you must, enable authentication and firewall rules.

- Use networks for isolation
  - Create an isolated network for the lab to prevent accidental cross-talk with other host services: e.g., `labnet`.

- Use .env for local overrides
  - Put non-secret configuration (ports, tags) in an `.env` file to make local overrides simple.

- Add healthchecks and restart policies (where useful)
  - Healthchecks help orchestrators and users know when a service is ready.
  - Example:
    ```
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    ```

- Manage sensitive data using Docker secrets or environment variable provisioning
  - Do not commit secrets to the repo. For interactive labs, prefer ephemeral credentials and document how to set them.

- depends_on vs readiness
  - `depends_on` only controls startup order, not service readiness. Use healthchecks and application-level retries to ensure connections succeed.

- Resource constraints (optional)
  - For predictable lab behavior, limit CPU/memory with `deploy.resources` (compose version caveats apply) or runtime flags.

- Clean up instructions
  - Document how to remove volumes and images safely to avoid leaving sensitive data on developer machines.

---

## Ports & Accessibility

Default example mappings (adjust to your config):
- Vulnerable Web App: host 8080 -> container 80 (http://localhost:8080) — from the original lab example.
- Application-stack frontend (new compose): host 80 -> container 80 (http://localhost) — note this differs from the lab example above.
- MongoDB: not mapped to the host in the lab example (container-only on labnet:27017); in the application stack MongoDB is internal and data is persisted to prod_data.
- Redis: internal-only cache (no host mapping by default).

Ensure:
- Only intended services are exposed to the host. Review compose files before running.

---

## Persistence & Data Management

- MongoDB in the examples uses a named Docker volume to persist data across container restarts:
  - lab example: mongo_data
  - application stack: prod_data
- To reset DB data:
  - Stop containers: docker compose down
  - Remove volume: docker volume rm <project>_prod_data or docker compose down --volumes
- To seed data:
  - Provide a seed script, use a Dockerfile to copy seeder into the vulnerable-app image, or run an init script in a container that can reach MongoDB on the internal network.
- Back up/restore:
  - Use `mongodump`/`mongorestore` against a running MongoDB in the lab network, or mount the volume to a temporary container for file-level access.

Migration considerations (if compose or volume names change):
- If a compose change renames a named volume, data will not automatically move. To migrate:
  - Stop containers.
  - Create a temporary container mounting both old and new volumes and copy data between them.
  - Verify file ownership/permissions after copy.
  - Restart services with the new compose.

---

## Security & Isolation Warnings

This environment intentionally includes insecure components. Follow these hard rules:

- Do not run this lab on production networks or systems connected directly to the internet.
- Prefer running the entire lab inside an isolated VM host (e.g., local VM) or an isolated network segment.
- Do not publish MongoDB (or admin ports) to public networks.
- Clean up volumes and containers after use to avoid persistent insecure data.
- Only allow authorized users to use this environment. Activities may be detectable and are potentially illegal if used against systems without authorization.

Specific note for recent compose change:
- The new MongoDB service is started without authentication in the default compose. This is convenient for local development but is insecure for production. If you expose MongoDB to a network, enable authentication and proper access control.
- Also note that the previous Postgres password present in the monitoring compose has been removed from the default compose, and the Postgres service itself was deleted.

Responsible use: Use this repository only for authorized testing, training, or research in controlled environments.

---

## Cleanup & Resource Management

Common commands:
- Stop: docker compose down
- Stop and remove volumes: docker compose down --volumes --remove-orphans
- Remove a single named volume: docker volume rm <volume_name>
- Remove images: docker rmi <image_name>

Monitor resource usage (CPU/memory) while running Kali and multiple services and increase host resources if containers are CPU/memory-starved.

---

## Troubleshooting

- Image pull failures:
  - Ensure you have network access and correct image names/tags.
  - Try docker pull <image> manually to view error details.

- Port conflicts:
  - If host port is already used, change the host port in docker-compose.yml.

- Containers start but app not running:
  - If product-catalogue or order-service reference base images only, they may start and then exit or remain idle. Add a CMD/command or mount source code and run the application process (see "Application stack" section for guidance).
  - If you added build contexts, run docker compose up --build to rebuild images.

- Dependency/connectivity issues:
  - Because depends_on does not wait for readiness, services may attempt to connect to DB/cache too early. Add healthchecks or wait-for scripts.

- Permission issues with volumes:
  - On Linux, adjust ownership or mount path permissions. Consider chown or using an entrypoint that fixes permission at container start.

- MongoDB connection problems:
  - Ensure the app uses the correct host name (db or mongodb depending on compose) and port (27017) on the shared network.
  - Check container logs for connection errors.

- Inspect logs:
  - docker compose logs -f <service>

If issues persist, consult test.md for additional notes and open an issue with logs and environment details.

---

## How to view the docker-compose diff locally

If you want to inspect the exact changes made to docker-compose.yml in a commit locally, run one of these commands from a git clone of the repository:

- Show the docker-compose.yml at a specific commit:
  - git show <commit-ish>:docker-compose.yml

- Show a diff between two commits:
  - git diff <old-commit> <new-commit> -- docker-compose.yml

- Show a single commit's changes to docker-compose.yml:
  - git show <commit> -- docker-compose.yml

Replace <commit> with the SHA or branch name. If you need a precise commit analysis, paste the output of one of these commands into an issue or pull request and the maintainers (or an automation) can review line-by-line.

---

## Credits, License & Responsible Use

- The repository includes offensive/target tooling for training and testing. Use responsibly.
- This README and test.md additions do not license or provide permission to perform unauthorized testing against third-party systems.
- Check the project LICENSE file for formal license information.

Legal disclaimer: The authors and maintainers are not responsible for misuse of the provided materials. You are solely responsible for ensuring your use complies with applicable laws and policies.

---

## Changelog

- 2026-05-06 — Documentation: Added test.md and README updates describing a small attack/defense lab topology (Kali attacker, Vulnerable Web App, MongoDB backend). Documented recommended network isolation, ports, persistence, quickstart example, and security warnings.
- 2026-05-06 — Documentation: Added a docker-compose best-practices checklist (pinning tags, volumes, healthchecks, .env usage, secrets advice) and guidance for inspecting compose diffs and migration considerations.
- 2026-05-06 — Compose change: Default docker-compose was updated from a monitoring/observability stack to an application stack (frontend-vcl, product-catalogue, order-service, db (MongoDB), cache (Redis)). Compose file version bumped to 3.9. Monitoring services (Prometheus, Grafana, Loki, node-exporter) and postgres-db were removed. Mongo data is persisted to a new named volume `prod_data`. The new app services reference base images (node/python) without build contexts or mounted source code — see README notes for how to make these runnable locally.

---

## Next Steps / Suggestions

- Provide a reproducible docker-compose.yml or IaC templates in the repo (if not already present).
- Add Dockerfiles and start commands (or mount source code) for product-catalogue and order-service so containers actually run the application code.
- Add a MongoDB seeder and web app source or build instructions.
- Add health checks and small automated integration tests to validate the topology in CI (in a controlled sandbox).
- Track images and versions used and add notes when updating vulnerable-app or DB images.
- If monitoring is still desired, consider adding a separate compose file (docker-compose.monitoring.yml) so monitoring and application concerns remain separate.

For more details, see test.md in this repository. If you would like, I can provide a tailored docker-compose.yml adapted exactly to the images used in this project — paste the image names/tags or the content of test.md and I will generate it.