# OpenMetadata Usage Guide

This guide covers the bundled Docker Compose deployment in this directory.

For copy-paste scenarios, see [EXAMPLES.md](EXAMPLES.md).
 For realistic catalog and governance data, see [DATA_EXAMPLES.md](DATA_EXAMPLES.md).

## 1. Start the stack

Prerequisites:

- Docker Desktop with Compose v2+
- At least 6 GB RAM and 4 vCPUs allocated to Docker

For a fresh setup:

```bash
cp .env.example .env
```

Set `FERNET_KEY` in `.env` to a unique valid Fernet key. For example, on a machine with the Python `cryptography` package:

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Start the services:

```bash
docker compose up -d
docker compose ps --all
```

The UI is available at <http://localhost:8585>. The admin health endpoint is available at <http://localhost:8586/healthcheck>.

The default host bindings are localhost-only. Database and Elasticsearch ports are available only inside the Compose network.

## 2. Log in and manage users

The default local administrator is:

- Username: `admin@open-metadata.org`
- Password: `admin`

Change this password immediately if the instance will be shared.

Self-signup is disabled by default. The recommended way to add users is:

1. Log in as the administrator.
2. Open **Settings → Members/Users**.
3. Add the user and assign the appropriate role.

To enable the signup form temporarily, set this in `.env`:

```env
AUTHENTICATION_ENABLE_SELF_SIGNUP=true
```

Apply the configuration with:

```bash
docker compose up -d
```

New users are not administrators automatically. Disable self-signup again after the required accounts are created.

## 3. Configure ingestion

The bundled ingestion service runs Airflow and is available at <http://localhost:8080>.

- Airflow username: `admin`
- Airflow password: `admin`

In OpenMetadata, create or select a data service, configure its connection, and create an ingestion workflow. You can run the workflow immediately or schedule it. Workflow-generated DAGs are stored in the persistent ingestion volumes.

The OpenMetadata server communicates with Airflow using:

```env
PIPELINE_SERVICE_CLIENT_ENDPOINT=http://ingestion:8080
SERVER_HOST_API_URL=http://openmetadata-server:8585/api
```

These are container-to-container URLs. If you change the host-facing OpenMetadata port, do not change these internal URLs.

## 4. Configuration rules

The main settings are in `.env`:

| Setting | Purpose |
|---|---|
| `OPENMETADATA_VERSION` | Version used for the server, database, and ingestion images |
| `OPENMETADATA_PORT` | Host port for the UI/API; container port remains 8585 |
| `OPENMETADATA_ADMIN_PORT` | Host port for the admin and health endpoint |
| `INGESTION_PORT` | Host port for Airflow |
| `OPENMETADATA_BIND_ADDRESS` | Host interface to bind; use `127.0.0.1` for local use |
| `OPENMETADATA_HEAP_OPTS` | JVM heap settings for the OpenMetadata server |
| `AUTHENTICATION_ENABLE_SELF_SIGNUP` | Enables or disables user self-registration |
| `FERNET_KEY` | Encrypts stored service connection secrets |

The bundled PostgreSQL service uses the fixed defaults `openmetadata_user`, `openmetadata_password`, and `openmetadata_db`. For custom database credentials, external databases, TLS, or production deployments, use OpenMetadata's external-database deployment configuration instead of modifying only these local defaults.

## 5. Common operations

View status:

```bash
docker compose ps --all
```

View logs:

```bash
docker compose logs -f openmetadata-server
docker compose logs -f ingestion
docker compose logs -f execute-migrate-all
```

Restart one service:

```bash
docker compose restart openmetadata-server
```

Stop and retain data:

```bash
docker compose stop
```

Start stopped services:

```bash
docker compose start
```

Remove containers while retaining named volumes:

```bash
docker compose down
```

Do not use `docker compose down -v` unless you intend to permanently delete metadata, search indexes, and ingestion state.

## 6. Persistence and backups

The stack stores data in these named Docker volumes:

- `openmetadata_postgres-data` — metadata and Airflow databases
- `openmetadata_elasticsearch-data` — search indexes
- `openmetadata_ingestion-volume-dag-airflow` — generated DAG configuration
- `openmetadata_ingestion-volume-dags` — DAG files
- `openmetadata_ingestion-volume-tmp` — ingestion temporary data

Before upgrades or destructive maintenance, stop new ingestion runs and back up PostgreSQL with native tools. For example:

```bash
docker compose exec -T postgresql pg_dump -U postgres openmetadata_db > openmetadata_db.sql
docker compose exec -T postgresql pg_dump -U postgres airflow_db > airflow_db.sql
```

Also preserve the Elasticsearch volume or plan to rebuild indexes after restoring the metadata database. Test restoration separately; a backup is useful only if it can be restored.

## 7. Upgrading

1. Read the OpenMetadata release compatibility notes.
2. Back up both PostgreSQL databases and persistent volumes.
3. Update `OPENMETADATA_VERSION` in `.env`.
4. Pull the new images:

```bash
docker compose pull
```

5. Recreate the stack:

```bash
docker compose up -d
```

The `execute-migrate-all` service runs the schema and search migrations before the server starts. Check its logs and confirm the server health endpoint before using the UI.

## 8. Troubleshooting

### The stack does not start

Check the dependency and migration logs:

```bash
docker compose ps --all
docker compose logs execute-migrate-all
docker compose logs openmetadata-server
```

### Compose reports that `FERNET_KEY` is missing

Set `FERNET_KEY` in `.env` to a valid generated key. An empty value or descriptive placeholder is not sufficient.

### The admin login fails

Use `admin@open-metadata.org`, not just `admin`. The Airflow login is separate and uses `admin` / `admin` by default.

### A changed UI port breaks ingestion

Keep `SERVER_HOST_API_URL` set to `http://openmetadata-server:8585/api` and `PIPELINE_SERVICE_CLIENT_ENDPOINT` set to `http://ingestion:8080`. These values use internal container ports.

### A host port is already in use

Change the relevant host port in `.env`, for example:

```env
OPENMETADATA_PORT=9000
```

The container-side port remains unchanged. Restart with `docker compose up -d`.

### A service remains unhealthy

Inspect its logs and health status:

```bash
docker compose ps --all
docker compose logs postgresql
docker compose logs elasticsearch
```

OpenMetadata is resource-intensive; ensure Docker has enough memory and CPU. Do not delete volumes as a first troubleshooting step.

## 9. Security practices

- Keep `OPENMETADATA_BIND_ADDRESS=127.0.0.1` unless remote access is required.
- Disable self-signup unless it is explicitly needed.
- Change the default OpenMetadata and Airflow passwords before sharing the instance.
- Use TLS and an external identity provider for network-accessible deployments.
- Generate a unique Fernet key per deployment and keep it with your secret backups.
- Do not commit `.env`, passwords, private keys, or database dumps.