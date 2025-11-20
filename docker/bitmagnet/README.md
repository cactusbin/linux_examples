# Bitmagnet Docker Compose Setup

*Bzzzzt* Your friendly neighborhood self-hosted BitTorrent indexer! *Ping*

## What the heck is Bitmagnet?

Bitmagnet is a **self-hosted BitTorrent indexer**, DHT crawler, content classifier, and torrent search engine with a slick web UI, GraphQL API, and Servarr stack integration. Think of it as your personal Pirate Bay that actually respects your privacy! Ayy lmao! [web:9][web:13]

## Architecture Overview

This setup uses the **binhex/arch-bitmagnet** image, which is an Arch Linux-based container that bundles:

- **Bitmagnet application** - The core indexer/crawler
- **Built-in PostgreSQL support** - Database integration 
- **Web UI** - Browse and search torrents
- **DHT crawler** - Discovers torrents from the BitTorrent DHT network
- **API** - GraphQL interface for programmatic access

## Quick Start

### Prerequisites

```bash
# Ensure you have Docker and Docker Compose installed
docker --version
docker compose version
```

### Basic Deployment

```bash
# Clone or navigate to the directory
cd docker/bitmagnet

# Create config and data directories
mkdir -p config data/postgres

# Optional: Create .env file for secrets
cat > .env << 'EOF'
POSTGRES_PASSWORD=your_secure_password_here
TMDB_API_KEY=your_tmdb_api_key_here
PUID=1000
PGID=1000
EOF

# Start the stack
docker compose up -d

# Check logs
docker compose logs -f bitmagnet
```

### Access the Web UI

Once running, access Bitmagnet at:

```
http://localhost:3333
```

The DHT crawler starts automatically and you should see torrents appear within 1-2 minutes! [web:13]

## Configuration Deep Dive

### Environment Variables

| Variable | Default | Description | Idiom |
|----------|---------|-------------|---------|
| `TMDB_API_KEY` | empty | TheMovieDB API key for metadata | Get from [TMDB](https://www.themoviedb.org/settings/api) |
| `POSTGRES_PASSWORD` | `postgres` | PostgreSQL password | **Change in production!** |
| `PROCESSOR_CONCURRENCY` | `4` | Parallel metadata processors | Set to `nproc - 1` for optimal performance |
| `POSTGRES_VACUUM_DB` | `false` | Run VACUUM on startup | Reclaims storage, run weekly |
| `POSTGRES_REINDEX_DB` | `false` | Reindex database on startup | Improves query speed |
| `PUID` | `99` | User ID for file ownership | Get with `id -u` |
| `PGID` | `100` | Group ID for file ownership | Get with `id -g` |
| `UMASK` | `022` | File creation mask | `000`=rwx, `022`=rx for group |

### Port Mappings

```yaml
ports:
  - "3333:3333"      # Web UI & GraphQL API
  - "3344:3344/tcp"  # BitTorrent DHT (TCP)
  - "3344:3344/udp"  # BitTorrent DHT (UDP) 
  - "5432:5432"      # PostgreSQL (optional external access)
```

**Idiom Note**: The DHT crawler requires **both TCP and UDP** on port 3344. This is how BitTorrent's Mainline DHT protocol works - it uses UDP for queries and TCP for data transfer. [web:13][web:14]

### Volume Mounts

```yaml
volumes:
  - ./config:/config                    # App configuration
  - ./data/postgres:/var/lib/postgresql/data  # Database files
  - /etc/localtime:/etc/localtime:ro   # Sync timezone
```

**Idiom**: Mounting `/etc/localtime` as read-only (`:ro`) ensures container logs use your system's timezone without allowing the container to modify host time config. Classic defensive Docker practice! [web:18]

## Usage Examples

### Starting/Stopping

```bash
# Start in foreground (watch logs)
docker compose up

# Start detached (background)
docker compose up -d

# Stop gracefully
docker compose down

# Stop and remove volumes (WARNING: deletes data!)
docker compose down -v
```

### Maintenance Tasks

```bash
# View logs
docker compose logs -f bitmagnet

# Execute psql in PostgreSQL container
docker compose exec bitmagnet-postgres psql -U postgres -d bitmagnet

# Check database size
docker compose exec bitmagnet-postgres \
  psql -U postgres -d bitmagnet -c \
  "SELECT pg_size_pretty(pg_database_size('bitmagnet'));"

# Manual VACUUM (reclaim space)
docker compose exec bitmagnet-postgres \
  psql -U postgres -d bitmagnet -c "VACUUM FULL;"

# Restart just bitmagnet service
docker compose restart bitmagnet
```

### Database Backup

```bash
# Backup database to file
docker compose exec -T bitmagnet-postgres \
  pg_dump -U postgres bitmagnet | \
  gzip > "bitmagnet-backup-$(date +%Y%m%d).sql.gz"

# Restore from backup
gunzip -c bitmagnet-backup-20251120.sql.gz | \
  docker compose exec -T bitmagnet-postgres \
  psql -U postgres bitmagnet
```

**Idiom**: The `-T` flag disables pseudo-TTY allocation, which is essential for piping data in/out of containers. Without it, you'll get weird binary garbage in your dumps! [web:13]

## Docker Compose Idioms Explained

### Service Dependencies

```yaml
depends_on:
  bitmagnet-postgres:
    condition: service_healthy
```

This uses Docker's **health check waiting** pattern. The `bitmagnet` service won't start until PostgreSQL passes its health check. Much better than sleep-based timing hacks! [web:13][web:14]

### Health Checks

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 20s
```

**Breaking it down**:
- `test`: Command to run (`pg_isready` checks PostgreSQL)
- `interval`: Check every 10 seconds
- `timeout`: Kill check if it takes > 5 seconds  
- `retries`: Try 5 times before marking unhealthy
- `start_period`: Grace period (20s) before failing checks count

This is the **standard PostgreSQL health check pattern**. [web:13][web:14]

### Shared Memory Size

```yaml
shm_size: 1g
```

PostgreSQL uses shared memory for caching and inter-process communication. The default 64MB is way too small for production workloads. 1GB is a reasonable starting point - adjust based on your database size and available RAM. [web:13][web:14]

### Network Isolation

```yaml
networks:
  bitmagnet-net:
    driver: bridge
```

Creates an **isolated bridge network**. Services on this network can communicate using service names as hostnames (e.g., `bitmagnet-postgres`), but external networks can't reach them unless ports are explicitly exposed. Good security practice! [web:13][web:14]

## Advanced Topics

### Using Environment File

Create `.env` file:

```bash
# .env file
POSTGRES_PASSWORD=super_secret_password
TMDB_API_KEY=abcdef123456789
PUID=1000
PGID=1000
PROCESSOR_CONCURRENCY=8
```

Docker Compose automatically loads `.env` files. Reference with `${VAR:-default}` syntax. [web:13][web:14]

### Disable DHT Crawler

If you only want to use Bitmagnet as a frontend for existing torrents (not crawling):

```yaml
# In docker-compose.yml, bitmagnet service
command:
  - worker
  - run  
  - --keys=http_server
  - --keys=queue_server
  # Remove: --keys=dht_crawler
```

### VPN Integration

For enhanced privacy, route through a VPN container:

```yaml
services:
  vpn:
    image: dperson/openvpn-client
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    volumes:
      - ./vpn:/vpn
    # ... VPN config ...
  
  bitmagnet:
    network_mode: "service:vpn"
    depends_on:
      - vpn
    # Remove 'ports' section - use VPN container's ports
```

[web:10][web:17]

## Troubleshooting

### Container won't start

```bash
# Check logs
docker compose logs

# Verify PostgreSQL health
docker compose ps

# Test database connection
docker compose exec bitmagnet-postgres pg_isready -U postgres
```

### Permission errors

```bash
# Fix ownership of mounted volumes
sudo chown -R $(id -u):$(id -g) config data

# Or run as root (less secure)
# Set PUID=0 and PGID=0 in .env
```

### Port conflicts

```bash
# Check what's using port 3333
sudo ss -tulpn | grep :3333

# Change host port in docker-compose.yml
ports:
  - "8080:3333"  # Now accessible on port 8080
```

### Database corruption

```bash
# Stop services
docker compose down

# Backup and remove data
mv data/postgres data/postgres.bak
mkdir -p data/postgres

# Restart (creates fresh DB)
docker compose up -d
```

## Performance Tuning

### PostgreSQL Tuning

Add to PostgreSQL environment:

```yaml
environment:
  # Increase connection limit
  POSTGRES_MAX_CONNECTIONS: 200
  # Increase shared buffers (25% of RAM)
  POSTGRES_SHARED_BUFFERS: 2GB
  # Increase work memory  
  POSTGRES_WORK_MEM: 50MB
```

### Processor Concurrency

```bash
# Optimal setting based on CPU cores
PROCESSOR_CONCURRENCY=$(nproc --all)
```

**Idiom**: `nproc` returns the number of CPU cores. Setting concurrency equal to cores maximizes throughput for CPU-bound tasks. [web:13]

## Security Considerations

1. **Change default passwords** - Don't use `postgres` in production
2. **Restrict port exposure** - Don't expose PostgreSQL (5432) publicly
3. **Use VPN** - Route DHT crawler through VPN for privacy  
4. **Regular updates** - Pull latest images weekly
5. **Backup strategy** - Automated daily backups

## References

- [Bitmagnet Official Site](https://bitmagnet.io)
- [Docker Hub - binhex/arch-bitmagnet](https://hub.docker.com/r/binhex/arch-bitmagnet)  
- [GitHub - bitmagnet-io/bitmagnet](https://github.com/bitmagnet-io/bitmagnet)
- [GitHub - binhex/arch-bitmagnet](https://github.com/binhex/arch-bitmagnet)
- [Unraid Support Forum](https://forums.unraid.net/topic/174999-support-binhex-bitmagnet)

---

*Whirrrr* Happy indexing, DJ! May your DHT crawler be swift and your torrents plentiful! *Swoosh* Ayy lmao! *Bzzzt*
