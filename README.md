# Gitea Docker Setup

Self-hosted Git service using Gitea with Docker Compose.

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
3. Edit `.env` with your specific configuration
4. Create required directories:
   ```bash
   mkdir -p /path/to/gitea/{data,config}
   ```
5. Start the container:
   ```bash
   docker-compose up -d
   ```
6. Access Gitea at `http://localhost:8181` (or your configured port)

## Configuration

### Environment Variables

All configuration is managed through the `.env` file. Key settings include:

**User & Permissions**
- `USER_UID` / `USER_GID`: Match these with your host user if using host volumes

**Database**
- `DB_TYPE`: Database type (sqlite3, mysql, postgres)

**Server Settings**
- `GITEA_DOMAIN`: Your domain name
- `GITEA_ROOT_URL`: Full URL with protocol
- `GITEA_HTTP_PORT`: External HTTP port

**SSH Settings**
- `GITEA_SSH_DOMAIN`: SSH domain
- `GITEA_SSH_PORT`: External SSH port (default: 2222)
- `GITEA_SSH_LISTEN_PORT`: Internal container SSH port (default: 22)

**Security**
- `DISABLE_REGISTRATION`: Set to `true` to disable public registration

**Repository Features**
- `ENABLE_PUSH_CREATE_USER`: Allow repository creation via push
- `ENABLE_PUSH_CREATE_ORG`: Allow org repository creation via push

### Volume Paths

Update the following paths in `.env` to match your host system:
- `GITEA_DATA_PATH`: Gitea data directory
- `GITEA_CONFIG_PATH`: Gitea configuration directory
- `TIMEZONE_PATH`: System timezone file
- `LOCALTIME_PATH`: System localtime file

## Usage

### Starting the Service
```bash
docker-compose up -d
```

### Viewing Logs
```bash
docker-compose logs -f
```

### Stopping the Service
```bash
docker-compose down
```

### Upgrading Gitea
1. Pull the latest image:
   ```bash
   docker-compose pull
   ```
2. Recreate the container:
   ```bash
   docker-compose up -d
   ```

### Backup

Important directories to backup:
- `${GITEA_DATA_PATH}`: Contains all repositories and data
- `${GITEA_CONFIG_PATH}`: Contains configuration files

```bash
# Example backup command
tar -czf gitea-backup-$(date +%Y%m%d).tar.gz /path/to/gitea/data /path/to/gitea/config
```

## Access

- **Web Interface**: `http://localhost:${GITEA_HTTP_PORT}` or `https://${GITEA_DOMAIN}`
- **SSH Clone**: `git clone ssh://git@${GITEA_SSH_DOMAIN}:${GITEA_SSH_PORT}/username/repo.git`
- **HTTP Clone**: `git clone https://${GITEA_DOMAIN}/username/repo.git`

## Initial Setup

On first run, Gitea will prompt you to complete the installation:
1. Navigate to the web interface
2. Configure database settings (SQLite3 by default)
3. Create the administrator account
4. Complete the installation

## Security Notes

- Registration is disabled by default (`DISABLE_REGISTRATION=true`)
- SSH runs on port 2222 to avoid conflicts with host SSH
- Keep your `.env` file secure and never commit it to version control
- Regularly backup your data directory
- Consider using a reverse proxy (nginx, Caddy, Traefik) for HTTPS

## Troubleshooting

### Permission Issues
If you encounter permission errors, ensure `USER_UID` and `USER_GID` in `.env` match the owner of your volume directories.

### SSH Connection Issues
- Verify `GITEA_SSH_PORT` is not blocked by firewall
- Check that the port isn't already in use
- Ensure port forwarding is correct if behind a router/firewall

### Cannot Access Web Interface
- Verify `GITEA_HTTP_PORT` is available
- Check container logs: `docker-compose logs gitea`
- Ensure `GITEA_ROOT_URL` matches your access method

## Documentation

- [Official Gitea Documentation](https://docs.gitea.com/)
- [Docker Installation Guide](https://docs.gitea.com/installation/with-docker)
- [Configuration Cheat Sheet](https://docs.gitea.com/administration/config-cheat-sheet)

## License

This configuration is provided as-is. Gitea is licensed under the MIT License.
