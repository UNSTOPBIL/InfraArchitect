# Vulnerable Node.js Product Catalogue Application

This repository contains a deliberately vulnerable Node.js web application backed by MongoDB and Redis, designed for security training, attack/defense exercises, and controlled lab environments.

## Overview

The default lab topology is built around:

- **Kali Linux** — attacker / analysis host
- **frontend-vcl** — exposed web frontend
- **product-catalogue** — application service
- **order-service** — application service
- **MongoDB (`db`)** — backend data store
- **Redis (`cache`)** — cache / supporting service

> **Security warning:** This stack is intentionally insecure in places and should **only** be used in isolated, authorized environments such as local VMs, private lab networks, or container sandboxes. Do **not** expose MongoDB or the vulnerable services to the public internet.

## Architecture

The lab is intended to model a small attack/defense environment where an attacker host interacts with a vulnerable application stack.

```text
[Kali Linux]
      |
      v
[frontend-vcl] -> [product-catalogue] -> [MongoDB]
                      |
                      v
               [order-service]
                      |
                      v
                    [Redis]
```

### Network and exposure model

- **frontend-vcl** is typically the only externally reachable application component.
- **MongoDB** should remain internal-only.
- **Redis** should remain internal-only.
- Application services communicate over an isolated Docker bridge network.
- Use named volumes for persistence where needed.

## Features

- Vulnerable web application for lab use
- MongoDB-backed data storage
- Redis cache support
- Docker Compose-based deployment
- Lab-friendly service separation
- Guidance for secure local isolation and testing

## Important Notes

- The stack is designed for **practice and experimentation**, not production.
- Some services may intentionally run without authentication or hardened defaults.
- The application should be treated as a target in a controlled environment only.
- Avoid reusing real credentials, secrets, or datasets.

## Docker Compose Notes

This repository uses Docker Compose to define the lab stack.

### Best practices

- Use **named volumes** for persistent data.
- Keep database services **off public interfaces**.
- Prefer an **isolated bridge network** for inter-service communication.
- Store environment-specific values in a `.env` file.
- Use **healthchecks** for services that depend on startup order.
- Remember that `depends_on` does **not** guarantee readiness.
- Never commit secrets or credentials to the repository.

### Compose version

The stack has been updated from **Compose 3.8 to 3.9**.

### Service stack changes

The previous monitoring-oriented stack was removed, including:

- Prometheus
- Grafana
- Loki
- node-exporter
- postgres-db

The new application stack includes:

- `frontend-vcl`
- `product-catalogue`
- `order-service`
- `db` (MongoDB)
- `cache` (Redis)

## Setup

### Prerequisites

- Docker Engine
- Docker Compose
- A Linux host, VM, or isolated lab environment
- Kali Linux or another attacker workstation for testing

### Start the stack

```bash
docker compose up -d
```

To view logs:

```bash
docker compose logs -f
```

To stop the stack:

```bash
docker compose down
```

To remove volumes as well:

```bash
docker compose down -v
```

## Configuration

Create a `.env` file for local overrides if needed. Example values may include:

- container image tags
- exposed ports
- database connection strings
- cache settings
- application secrets

Example:

```env
MONGO_URI=mongodb://db:27017/products
REDIS_URL=redis://cache:6379
```

## Service Implementation Notes

The documentation for this lab highlights an important implementation detail:

- `product-catalogue` and `order-service` use base images only
- there are no build contexts, mounted source code, or run commands
- as a result, these containers may start without actually running application code

If you are extending this stack, verify that each service includes:

- a valid `build` context or application image
- the correct startup command
- required environment variables
- reachable dependencies
- healthchecks where appropriate

## Security Considerations

Because this repository is used for a vulnerable lab environment:

- Do not expose services to the public internet.
- Do not use production credentials.
- Keep MongoDB unauthenticated only if that is explicitly required for the exercise.
- Run the stack only in controlled, authorized environments.
- Isolate the lab from sensitive networks and systems.

## Troubleshooting

### Application containers exit immediately

- Check whether the image includes a valid startup command.
- Confirm that the service has a build context or a runnable base image.
- Review logs with `docker compose logs -f`.

### MongoDB connection failures

- Verify the `MONGO_URI` points to the correct hostname.
- Ensure the `db` container is on the same network as the application services.
- Confirm that MongoDB is fully initialized before the app starts.

### Redis connection failures

- Verify the `REDIS_URL` or equivalent configuration.
- Ensure the `cache` service is reachable on the internal network.

### Port conflicts

- Check whether another service is already listening on the exposed host ports.
- Adjust published ports in the Compose file or `.env`.

### Volume or permission issues

- Confirm Docker has permission to create and mount volumes.
- Remove stale volumes if initialization data becomes inconsistent.

## Migration Notes

This repository has shifted away from a monitoring stack toward a vulnerable application lab. If you are updating an older environment:

- remove references to Prometheus, Grafana, Loki, node-exporter, and postgres-db
- update service references to the new application topology
- verify any dashboards or monitoring scripts are no longer required
- review network assumptions and exposed ports

## License

This project is intended for educational and lab use. Refer to the repository’s license information if provided.