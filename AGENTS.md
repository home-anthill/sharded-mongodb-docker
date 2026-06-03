# AGENTS.md

This file provides guidance to coding agents when working with code in this repository.

## Project Overview

Docker Compose project (part of the [home-anthill](https://github.com/home-anthill) ecosystem) that deploys a MongoDB Sharded Cluster locally. This is a **custom fork** of [pkdone/sharded-mongodb-docker](https://github.com/pkdone/sharded-mongodb-docker) **with reduced replica counts** (2 replicas per set instead of 3). The project was renamed from `sharded-mongodb-docker` to `sharded-mongodb-compose` to better reflect its nature as a Compose-based deployment.

The cluster consists of 6 containers:
- **Shard0**: 2-member replica set (`shard0-replica0`, `shard0-replica1`)
- **ConfigDB**: 2-member replica set (`configdb-replica0`, `configdb-replica1`)
- **Mongos routers**: 2 query routers (`mongos-router0` on host port 27017, `mongos-router1` on host port 27018)

## Prerequisites

- Docker Desktop with `docker compose` command
- MongoDB Shell (`mongosh`) installed on your workstation, or MongoDB Compass for GUI access

## File Structure

```
sharded-mongodb-compose/
├── docker-compose.yml       # Defines all 6 containers and their configuration
├── mongod/
│   ├── Dockerfile          # Custom mongod image entrypoint
│   ├── mongod-start.sh     # Entrypoint that forks runextra.sh then execs official entrypoint
│   ├── mongod-runextra.sh  # Background init script (runs rs.initiate() if DO_INIT_REPSET=true)
│   └── mongod.conf         # Shared MongoDB server config (mounted as volume in all mongod containers)
└── mongos/
    ├── Dockerfile          # Custom mongos image entrypoint
    ├── mongos-start.sh     # Entrypoint that forks runextra.sh then execs official entrypoint
    └── mongos-runextra.sh  # Background init script (runs sh.addShard() on router0 only)
```

## Commands

Docker is the suggested container runtime.

```bash
# Build and start the cluster
docker compose up --build -d

# Stop and remove all containers
docker compose down

# Check running containers
docker compose ps

# View logs for a specific container
docker compose logs mongos-router0

# Connect to the cluster via mongosh
mongosh --port 27017

# Verify cluster status (from mongosh shell)
sh.status()
```

## Architecture

### Container Startup Flow

Each container uses a two-phase entrypoint pattern. All scripts are POSIX `/bin/sh` (not bash).

1. **`*-start.sh`** (entrypoint): Forks a background `*-runextra.sh` script, then `exec`s Docker's official `docker-entrypoint.sh`.
2. **`*-runextra.sh`** (background init):
   - **mongod**: If `DO_INIT_REPSET=true`, waits for mongod to be ready, then calls `rs.initiate()` to bootstrap the replica set. Only `replica0` of each set has this flag.
   - **mongos**: Exits immediately if `SHARD_LIST` is unset. Otherwise, waits for mongos to be ready, then calls `sh.addShard()` for each shard in `SHARD_LIST` (semicolon-delimited). Only `mongos-router0` has `SHARD_LIST` set.

### Key Environment Variables (set in docker-compose.yml)

- `REPSET_NAME`: Replica set name used by `mongod-runextra.sh` for `rs.initiate()` (e.g., `shard0`, `configdb`)
- `DO_INIT_REPSET`: When `true`, triggers replica set initialization on that container (only set on `replica0` of each set)
- `SHARD_LIST`: Semicolon-delimited shard connection strings consumed by `mongos-runextra.sh` (only set on `mongos-router0`)

### Replica Set Details

The cluster uses **2-member replica sets** (instead of the upstream project's 3-member sets) to support MongoDB transactions while minimizing resource usage. This is suitable for local development:

- **Shard0** replica set holds the sharded data
- **ConfigDB** replica set holds the cluster configuration (required for sharding)
- Each replica set needs at least 2 members to form a quorum and support multi-document transactions

### Networking

All containers share the `internalnetwork` Docker network. Container hostnames match their service names (e.g., `shard0-replica0`). Only mongos routers expose ports to the host, bound to `127.0.0.1` (localhost only, not accessible from LAN).

### Security

This cluster is intended for **local development only** and has no authentication or TLS enabled. The mongos router ports are bound to `127.0.0.1` to prevent exposure to the local network.

## Troubleshooting

**Containers fail to start or exit immediately:**
- Check Docker daemon is running: `docker ps`
- Review logs for the failed container: `docker compose logs <container_name>`
- Ensure port 27017/27018 are not already in use on your host

**Cannot connect with mongosh:**
- Verify containers are running: `docker compose ps`
- Check that mongosh is installed: `mongosh --version`
- Try connecting to the second router if the first fails: `mongosh --port 27018`

**Replica set initialization fails:**
- The init scripts run in the background; initialization may take 10-30 seconds
- Check the mongod/mongos logs for "rs.initiate()" or "sh.addShard()" output
- Verify all containers are healthy: `docker compose ps` should show all as "Up"

**Port already in use:**
- If 27017/27018 are occupied, modify the port mappings in `docker-compose.yml`
- Or stop conflicting services: `docker compose down && lsof -i :27017`
