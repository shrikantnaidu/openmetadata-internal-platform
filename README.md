# OpenMetadata — Docker Stack

Self-hosted OpenMetadata 1.12.6 with the bundled PostgreSQL database, Elasticsearch search engine, schema migration job, OpenMetadata server/UI, and ingestion service.

## Architecture

```text
openmetadata/
├── postgresql             # Metadata and ingestion databases
├── elasticsearch          # Search index storage
├── execute-migrate-all    # One-shot schema/index migration
├── openmetadata-server    # OpenMetadata API and UI
└── ingestion              # Airflow-based metadata ingestion service
```

The stack is independent from the sibling Airflow, MLflow, and Feast stacks. Data is kept in named Docker volumes and the database/search ports are private to the Compose network.

## Prerequisites

- Docker CE with Docker Compose v2+
- At least 6 GB RAM and 4 vCPUs available to Docker

OpenMetadata’s local deployment is resource-intensive because it runs PostgreSQL, Elasticsearch, the server, and ingestion together.

## Quick Start

### 1. Configure environment

The included `.env` works for a local test instance. The bundled quickstart uses `admin@open-metadata.org` / `admin`; keep the default ports bound to localhost and do not expose those credentials to an untrusted network. For a production deployment, use externally managed database/search services and a supported authentication provider. To create a template for another deployment:

```bash
cp .env.example .env
```

After copying the example, set `FERNET_KEY` to a unique generated Fernet key before starting the stack. Changing `OPENMETADATA_PORT` only changes the host-side mapping; leave `AUTHENTICATION_PUBLIC_KEYS` and `SERVER_HOST_API_URL` at their container-reachable defaults. If a reverse proxy or external hostname is used, configure URLs that are reachable from the containers rather than using `localhost`.

The default host bindings are localhost-only. To intentionally expose the stack on a network, set `OPENMETADATA_BIND_ADDRESS` to an appropriate interface, disable the default credentials, and configure TLS and authentication first.

### 2. Start the stack

```bash
docker compose up -d
```

The migration container runs once before the server starts. Inspect startup status with:

```bash
docker compose ps
docker compose logs -f openmetadata-server
```

OpenMetadata UI/API: [http://localhost:8585](http://localhost:8585)

Default local login: `admin@open-metadata.org` / `admin`.

### 3. Verify health

```bash
curl http://localhost:8586/healthcheck
```

The health endpoint should return a successful response after migration completes and the server is ready.

## Usage guide

See [USAGE.md](USAGE.md) for setup, user management, ingestion, backups, upgrades, and troubleshooting.

See [EXAMPLES.md](EXAMPLES.md) for common setup and operation scenarios.

See [DATA_EXAMPLES.md](DATA_EXAMPLES.md) for catalog, governance, lineage, glossary, tagging, and data-quality examples.

## Useful commands

| Task | Command |
|---|---|
| Stop services | `docker compose stop` |
| Start stopped services | `docker compose start` |
| View logs | `docker compose logs -f` |
| Restart server | `docker compose restart openmetadata-server` |
| Run migrations again | `docker compose run --rm execute-migrate-all` |
| Stop and remove containers | `docker compose down` |
| Stop and delete all data | `docker compose down -v` |

The last command permanently removes metadata, search indexes, and ingestion state stored in the named volumes.

## Production notes

This bundled stack is intended for local, internal, and small self-hosted deployments. For production, OpenMetadata recommends using externally managed PostgreSQL and Elasticsearch/OpenSearch services, TLS, a strong authentication setup, backups, and resource sizing appropriate to the catalog workload.

The Compose definition is adapted from the official OpenMetadata 1.12.6 PostgreSQL deployment:

<https://github.com/open-metadata/OpenMetadata/releases/download/1.12.6-release/docker-compose-postgres.yml>
