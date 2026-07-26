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

Postgres credentials are shared between the `db` container and the `gitea` container via `.env`. `POSTGRES_PASSWORD` is required — the stack refuses to start without it.

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

   ```bash
   GITEA_SIGNING_KEY=/data/git/.ssh/gitea-signing.pub
   GITEA_SIGNING_NAME=Gitea
   GITEA_SIGNING_EMAIL=gitea@your.domain
   GITEA_SIGNING_INITIAL_COMMIT=always
   ```

3. Restart: `docker compose up -d`

## Migrating from SQLite to PostgreSQL

If you're coming from the old SQLite-based setup, migrate the DB before switching. The reliable pattern is **schema-first, data-only**: let Gitea build its correct Postgres schema against an empty DB, then use [`pgloader`](https://pgloader.readthedocs.io/) to load only the data.

> ⚠️ **Do NOT run pgloader in its default mode against Gitea.** It will mirror SQLite's `sqlite_autoindex_*` UNIQUE constraints as bare indexes, and Gitea's Postgres migrations will get stuck trying to drop them (`ERROR: cannot drop index ... because constraint ... requires it`). Use the schema-first workflow below.

**Before you start:** back up your existing `data/` directory. Gitea stores its SQLite DB at `data/gitea/gitea.db`.

1. **Stop the old stack:**

   ```bash
   docker compose down
   ```

2. **Copy the SQLite DB somewhere pgloader can reach:**

   ```bash
   cp ${DOCKER_DATA_PATH}/data/gitea/gitea.db /tmp/gitea.db
   chmod 644 /tmp/gitea.db
   ```

3. **Bring up an empty Postgres, then let Gitea create its schema:**

   ```bash
   docker compose up -d db
   docker compose ps                       # wait for db to be healthy
   docker compose up -d gitea
   docker compose logs -f gitea            # watch until "Starting new Web server"
   # Ctrl+C once it's up
   ```

4. **Stop just Gitea so pgloader has exclusive DB access** (keep `db` running):

   ```bash
   docker compose stop gitea
   ```

5. **Run pgloader in data-only mode** (on Synology DSM prepend `sudo` — DSM users aren't in a `docker` group). Confirm the network name first with `docker network ls | grep gitea`:

   ```bash
   docker run --rm --network gitea_gitea \
     -v /tmp/gitea.db:/tmp/gitea.db \
     dimitri/pgloader:latest \
     pgloader --with "data only" \
              --with "disable triggers" \
              --with "truncate" \
              sqlite:///tmp/gitea.db \
              postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
   ```

   If it complains about foreign keys, add `--with "concurrency = 1"` and rerun. `--with "truncate"` empties each table before inserting, so the load is idempotent — safe to rerun.

6. **Reset the sequences.** SQLite has no sequences; every Postgres serial column now starts at 1, which collides with existing IDs on the next insert. Fix in one shot:

   ```bash
   docker exec -i gitea-db psql -U ${POSTGRES_USER} -d ${POSTGRES_DB} <<'SQL'
   DO $$
   DECLARE
       r RECORD;
   BEGIN
       FOR r IN
           SELECT c.relname AS seq, t.relname AS tbl, a.attname AS col
           FROM pg_class c
           JOIN pg_depend d ON d.objid = c.oid
           JOIN pg_class t ON t.oid = d.refobjid
           JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = d.refobjsubid
           WHERE c.relkind = 'S'
       LOOP
           EXECUTE format('SELECT setval(%L, COALESCE((SELECT MAX(%I) FROM %I), 0) + 1, false)',
                          r.seq, r.col, r.tbl);
       END LOOP;
   END $$;
   SQL
   ```

7. **Start Gitea:**

   ```bash
   docker compose up -d gitea
   docker compose logs -f gitea
   ```

8. **Verify:** log in, browse a few repos, check user accounts, webhooks, and issue history. Try creating a new issue/PR — if sequences weren't reset properly, you'll hit a duplicate-key error and know to rerun step 6. Once everything looks good, remove `/tmp/gitea.db` and (once you've confirmed no rollback is needed) the old `data/gitea/gitea.db`.

### Recovering from a failed migration

If Gitea is stuck in a restart loop with `cannot drop index ... because constraint ... requires it`, you loaded schema-first-not-followed and need to redo it:

```bash
docker compose down
sudo rm -rf ${DOCKER_DATA_PATH}/postgres        # wipe the mangled DB
```

Then restart from step 3 above.

### Fallback

If `pgloader` still fails, Gitea's [built-in `dump` / restore](https://docs.gitea.com/administration/backup-and-restore) can migrate — it exports the full instance (DB + repos + config) to a zip, and you restore it against a fresh install pointed at Postgres. Slower but more forgiving.

## Deploying with Komodo

This stack is set up for [Komodo](https://komo.do) deployments:

- The compose file uses `env_file: .env` plus a full `environment:` block, so Komodo can either mount `.env` from disk or inject each variable via its Variables UI.
- `${POSTGRES_PASSWORD:?...}` fails fast if the password isn't set — Komodo surfaces the error immediately instead of starting a broken stack.
- `DOCKER_DATA_PATH` is the single knob for where all state lives; point it at the host path Komodo manages for this stack.
- `komodo_ui_compose.yaml` and `komodo.env` in this repo are the live production files — reference them for the exact shape Komodo expects.

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

### `docker: permission denied ... /var/run/docker.sock` (Synology DSM)

DSM doesn't put non-`root` users in a `docker` group. Prepend `sudo` to any `docker` command, or add your user to the `docker` group (Package Center → Container Manager → User permissions).

### `cannot drop index ... because constraint ... requires it`

You ran `pgloader` in its default mode against Gitea. See [Recovering from a failed migration](#recovering-from-a-failed-migration) above.

## Documentation

- [Gitea Documentation](https://docs.gitea.com/)
- [Docker Installation Guide](https://docs.gitea.com/installation/install-with-docker)
- [Config Cheat Sheet](https://docs.gitea.com/administration/config-cheat-sheet)
- [Backup and Restore](https://docs.gitea.com/administration/backup-and-restore)
- [Signing Commits](https://docs.gitea.com/administration/signing)

## License

This configuration is provided as-is. Gitea is licensed under the MIT License.
