# OpenMetadata Setup Examples

These examples assume the bundled Compose stack in this directory.

## Example 1: First local startup

Create the environment file:

```bash
cp .env.example .env
```

Set a valid Fernet key in `.env`, then start the stack:

```bash
docker compose up -d
docker compose ps --all
```

Check that the server is ready:

```bash
curl http://localhost:8586/healthcheck
```

Open <http://localhost:8585> and sign in with:

```text
Username: admin@open-metadata.org
Password: admin
```

Change the password after the first login.

## Example 2: Use a different UI port

To make the UI available on port 9000, change only the host-side port:

```env
OPENMETADATA_PORT=9000
```

Restart the stack:

```bash
docker compose up -d
```

Open <http://localhost:9000>.

Keep these values unchanged because they are container-to-container URLs:

```env
SERVER_HOST_API_URL=http://openmetadata-server:8585/api
PIPELINE_SERVICE_CLIENT_ENDPOINT=http://ingestion:8080
```

## Example 3: Add another user without self-signup

Self-signup is disabled by default. Add users through the UI:

1. Sign in as `admin@open-metadata.org`.
2. Open **Settings → Members/Users**.
3. Select **Add User**.
4. Enter the user email and assign a role.
5. Ask the user to sign in and change their password.

This is the preferred approach for an internal catalog because administrators control account creation.

## Example 4: Temporarily enable self-signup

Edit `.env`:

```env
AUTHENTICATION_ENABLE_SELF_SIGNUP=true
```

Apply the change:

```bash
docker compose up -d
```

Users can now use the signup option on the login page. After the required accounts are created, disable it again:

```env
AUTHENTICATION_ENABLE_SELF_SIGNUP=false
```

Then apply the change:

```bash
docker compose up -d
```

New accounts are regular users unless an administrator assigns elevated permissions.

## Example 5: Create and run an ingestion workflow

A typical workflow looks like this:

1. Open OpenMetadata at <http://localhost:8585>.
2. Create a data service, such as PostgreSQL, MySQL, or a warehouse service.
3. Enter the source connection details and test the connection.
4. Open the service's **Ingestion** section.
5. Create a metadata ingestion workflow.
6. Choose **Run Now** for a one-time test or add a schedule.
7. Check the workflow status and logs in OpenMetadata or Airflow at <http://localhost:8080>.

The bundled server reaches Airflow through `http://ingestion:8080`; the browser reaches Airflow through the host mapping at `http://localhost:8080`.

## Example 6: Investigate a failed migration

If the server does not start, inspect the migration first:

```bash
docker compose ps --all
docker compose logs execute-migrate-all
docker compose logs openmetadata-server
```

A successful migration exits with status 0. The server starts only after that completion condition is satisfied.

## Example 7: Back up metadata before maintenance

Export both PostgreSQL databases before an upgrade or destructive maintenance:

```bash
docker compose exec -T postgresql pg_dump -U postgres openmetadata_db > openmetadata_db.sql
docker compose exec -T postgresql pg_dump -U postgres airflow_db > airflow_db.sql
```

Keep the SQL files secure. Also preserve the named Elasticsearch volume or plan to rebuild search indexes after restoring the metadata database.

## Example 8: Stop the stack without deleting data

Stop services while retaining all named volumes:

```bash
docker compose stop
```

Start them again later:

```bash
docker compose start
```

Use this only when you intend to delete all local data:

```bash
docker compose down -v
```

## Example 9: Allow access from another machine

The default binding is local-only:

```env
OPENMETADATA_BIND_ADDRESS=127.0.0.1
```

For controlled network access, change it to an appropriate host interface, for example:

```env
OPENMETADATA_BIND_ADDRESS=0.0.0.0
```

Before doing this, change default passwords, keep self-signup disabled, configure TLS or a reverse proxy, and restrict firewall access. Do not expose the default `admin/admin` credentials to a network.