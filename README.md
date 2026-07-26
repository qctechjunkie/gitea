# Gitea Docker Setup

Self-hosted Git service using [Gitea](https://about.gitea.com/) with Docker Compose and PostgreSQL.

## Prerequisites

- Docker
- Docker Compose
- Domain name configured (optional but recommended)

## Quick Start

1. Clone or copy this repository
2. Copy the example environment file and configure it:
   ```bash
   cp .env.example .env
   ```
3. Edit `.env` — at minimum set `DOCKER_DATA_PATH` and `POSTGRES_PASSWORD`
4. Create the data directories (owned by the UID/GID from `.env`):
   ```bash
   mkdir -p ${DOCKER_DATA_PATH}/{data,postgres}
   chown -R ${USER_UID}:${USER_GID} ${DOCKER_DATA_PATH}
   ```
5. Start the stack:
   ```bash
   docker compose up -d
   ```
6. Access Gitea at `http://<host>:${GITEA_HTTP_PORT}` and complete the first-run setup

## Configuration

All configuration is managed through the `.env` file. See `.env.example` for the full list with inline documentation.

### Volume Paths

- `DOCKER_DATA_PATH` — base directory on the host. The compose file appends:
  - `${DOCKER_DATA_PATH}/data` → `/data` (Gitea app data, repos, config, avatars)
  - `${DOCKER_DATA_PATH}/postgres` → `/var/lib/postgresql/data` (database)

### Image Tags

- `GITEA_TAG` — defaults to `1.27` (tracks the latest 1.27.x). Pin to a full version like `1.27.0` for zero-surprise updates.
- `POSTGRES_TAG` — defaults to `17`. Gitea supports Postgres 12+; 17 is current stable.

### Ports

- `GITEA_HTTP_PORT` (default `3000`) — web UI
- `GITEA_SSH_PORT` (default `2222`) — git-over-ssh; container listens on `22` internally

### Database

Postgres credentials are shared between the `db` container and the `server` container via `.env`. `POSTGRES_PASSWORD` is required — the stack refuses to start without it.

### User / Permissions

`USER_UID` and `USER_GID` must match the owner of `${DOCKER_DATA_PATH}` on the host, or Gitea will fail to start with permission errors.

## Usage

### Starting the service
```bash
docker compose up -d
```

### Viewing logs
```bash
docker compose logs -f
```

### Stopping the service
```bash
docker compose down
```

### Upgrading
Bump `GITEA_TAG` in `.env` (or leave it on the major and let patches roll in), then:
```bash
docker compose pull
docker compose up -d
```
Read the Gitea [release notes](https://blog.gitea.com/) before crossing a minor version.

### Backup

Stop the stack (or use `pg_dump` for a hot Postgres backup), then archive both volumes:
```bash
docker compose down
tar -czf gitea-backup-$(date +%Y%m%d).tar.gz -C ${DOCKER_DATA_PATH} data postgres
docker compose up -d
```

Hot backup (no downtime):
```bash
docker compose exec -T db pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB} \
  | gzip > gitea-db-$(date +%Y%m%d).sql.gz
```

## SSH Commit Signing

This stack enables Gitea's SSH signature verification (Gitea 1.24+). Users can sign commits with an SSH key and Gitea will verify the signature and display a green **Verified** badge in the web UI.

### Setup per-user (recommended)

1. On your workstation, use an existing SSH key or generate one:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_signing -C "your@email"
   ```
2. Configure git to sign with SSH:
   ```bash
   git config --global gpg.format ssh
   git config --global user.signingkey ~/.ssh/id_ed25519_signing.pub
   git config --global commit.gpgsign true
   git config --global tag.gpgsign true
   ```
3. In Gitea → **Settings → SSH / GPG Keys → Add Key**, paste the *public* key and check **Use as signing key** (this is the important part — without it, Gitea won't accept the key for verification).
4. Commit → push. The commit shows **Verified** in the Gitea UI.

The verification policies (`GITEA_SIGNING_CRUD_ACTIONS`, `GITEA_SIGNING_MERGES`, etc.) control which server-generated commits require signatures. Defaults require signed parent commits + a pubkey on file for CRUD actions and a fully signed history for merges. See the [signing docs](https://docs.gitea.com/administration/signing) for the full grammar.

### Server-side signing (optional)

To have **Gitea itself** sign initial commits, wiki edits, and merges (rather than just verifying user commits), set `GITEA_SIGNING_KEY` to a path inside the container instead of `none`:

1. Generate a keypair on the host:
   ```bash
   ssh-keygen -t ed25519 -f ${DOCKER_DATA_PATH}/data/git/.ssh/gitea-signing -N ""
   chown -R ${USER_UID}:${USER_GID} ${DOCKER_DATA_PATH}/data/git/.ssh
   ```
2. In `.env` (or Komodo Variables):
   ```
   GITEA_SIGNING_KEY=/data/git/.ssh/gitea-signing.pub
   GITEA_SIGNING_NAME=Gitea
   GITEA_SIGNING_EMAIL=gitea@your.domain
   GITEA_SIGNING_INITIAL_COMMIT=always
   ```
3. Restart: `docker compose up -d`

## Migrating from SQLite to PostgreSQL

If you're coming from the old SQLite-based setup, migrate the DB before switching. The cleanest path is [`pgloader`](https://pgloader.readthedocs.io/), which handles schema + data in one shot.

**Before you start:** back up your existing `data/` directory. Gitea stores its SQLite DB at `data/gitea/gitea.db`.

1. **Stop the old Gitea container** (this ensures no writes are in flight):
   ```bash
   docker compose down
   ```

2. **Copy the SQLite DB somewhere accessible** — e.g. into `${DOCKER_DATA_PATH}` so it lands next to the new setup:
   ```bash
   cp ${DOCKER_DATA_PATH}/data/gitea/gitea.db /tmp/gitea.db
   ```

3. **Start ONLY the new Postgres container** (the `server` will fail-and-restart while the DB is empty — that's fine):
   ```bash
   docker compose up -d db
   ```

4. **Wait for the DB to be healthy:**
   ```bash
   docker compose ps
   ```

5. **Run pgloader from a temporary container** — no host install needed. On Synology DSM (and any host where your user isn't in a `docker` group), prepend `sudo`:
   ```bash
   docker run --rm --network gitea_gitea \
     -v /tmp/gitea.db:/tmp/gitea.db \
     dimitri/pgloader:latest \
     pgloader sqlite:///tmp/gitea.db \
       postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
   ```
   > The network name depends on how the stack was imported — `<project>_gitea` for docker compose, but Komodo may name it `<stack>_default` or similar. Confirm with `docker network ls | grep gitea`.
   >
   > If Gitea has ever booted against this Postgres and left tables behind, pgloader will error on constraint conflicts. Drop and recreate the DB first:
   > ```bash
   > docker exec -i gitea-db psql -U ${POSTGRES_USER} -d postgres \
   >   -c "DROP DATABASE ${POSTGRES_DB};" \
   >   -c "CREATE DATABASE ${POSTGRES_DB} OWNER ${POSTGRES_USER};"
   > ```

6. **Start Gitea:**
   ```bash
   docker compose up -d
   ```

7. **Verify:** log in, browse a few repos, check user accounts, webhooks, and issue history. Once you're satisfied, remove `/tmp/gitea.db` and the old `data/gitea/gitea.db` file (leaving Gitea's other files under `data/` in place).

**Fallback:** if `pgloader` fails, Gitea's [built-in `dump` / restore](https://docs.gitea.com/administration/backup-and-restore) can migrate too, but it requires exporting the full instance, wiping, and restoring with the new DB. `pgloader` is simpler when it works.

## Deploying with Komodo

This stack is set up for [Komodo](https://komo.do) deployments:

- The compose file uses `env_file: .env` plus a full `environment:` block, so Komodo can either mount `.env` from disk or inject each variable via its Variables UI.
- `${POSTGRES_PASSWORD:?...}` fails fast if the password isn't set — Komodo surfaces the error immediately instead of starting a broken stack.
- `DOCKER_DATA_PATH` is the single knob for where all state lives; point it at the host path Komodo manages for this stack.

Import as a **Stack** in Komodo, wire up `.env` (or add each variable in the Variables tab), and deploy.

## Access

- **Web**: `http://<host>:${GITEA_HTTP_PORT}`
- **SSH clone**: `git clone ssh://git@<host>:${GITEA_SSH_PORT}/user/repo.git`
- **HTTP clone**: `git clone http://<host>:${GITEA_HTTP_PORT}/user/repo.git`

Put a reverse proxy (nginx-proxy-manager, Caddy, Traefik) in front for HTTPS.

## Initial Setup

On first run, Gitea shows an install page:
1. Navigate to the web UI
2. Database settings are already populated from `.env` — just confirm
3. Create the administrator account
4. Complete the install

If you migrated with pgloader, Gitea will pick up the existing DB and skip the install page — go straight to login.

## Security Notes

- Never commit `.env` — it contains the DB password
- SSH is exposed on a non-standard host port (`2222` by default) to avoid conflicts with host SSH
- Put HTTPS in front via a reverse proxy for anything reachable outside your LAN
- Back up `${DOCKER_DATA_PATH}` regularly

## Troubleshooting

### Container won't start / permission errors
Ensure `USER_UID` and `USER_GID` in `.env` match the owner of `${DOCKER_DATA_PATH}`:
```bash
ls -ln ${DOCKER_DATA_PATH}
```

### Cannot connect to database
- Check that `POSTGRES_PASSWORD` is set in `.env`
- Confirm the `db` container is healthy: `docker compose ps`
- Check logs: `docker compose logs db`

### SSH not reachable
- Verify the host port matches `GITEA_SSH_PORT` and is open in your firewall
- Confirm the port isn't already bound: `ss -tlnp | grep :${GITEA_SSH_PORT}`

### pgloader migration errors
- `Connection refused`: the Postgres container isn't ready yet — wait for `docker compose ps` to show it healthy
- `network not found`: check the actual network name with `docker network ls | grep gitea`
- `constraint violation`: pgloader occasionally trips on Gitea's foreign keys; re-run with `--with "data only"` after first creating the schema by letting Gitea start against an empty DB, then wiping it

## Documentation

- [Gitea Documentation](https://docs.gitea.com/)
- [Docker Installation Guide](https://docs.gitea.com/installation/install-with-docker)
- [Config Cheat Sheet](https://docs.gitea.com/administration/config-cheat-sheet)
- [Backup and Restore](https://docs.gitea.com/administration/backup-and-restore)

## License

This configuration is provided as-is. Gitea is licensed under the MIT License.
