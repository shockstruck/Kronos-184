# Kronos-184: Containerization Assessment

## Project Overview

Kronos is an OSRS (Old School RuneScape) private game server. It is a Java 21 multi-module Gradle project using Kotlin 2.1.21, Lombok, and Netty for networking.

## Components

| Component | Port | Containerize? | Notes |
|---|---|---|---|
| `kronos-server` | 13302 | **Yes** | Main game server, primary target |
| `kronos-update-server` | 7304 | **Yes** | Cache update server, separate container |
| `kronos-webhooks` | - | **Maybe** | Discord integration, future candidate |
| `runelite` (client) | - | **No** | Desktop GUI app, not suitable |
| `kronos-launcher` | - | **No** | Client launcher, not a server |
| `Kronos-web` (PHP) | - | **Maybe** | Simple PHP scripts, future candidate |

## Container Architecture

Both server containers are built in the [docker-containers](https://github.com/shockstruck/docker-containers) repository following standardized build patterns:

```
docker-containers/apps/
  kronos-server/
    Dockerfile          # Multi-stage JDK 21 build
    docker-bake.hcl     # Version pinning and OCI labels
    tests.yaml          # Container structure tests
  kronos-update-server/
    Dockerfile          # Multi-stage JDK 21 build
    docker-bake.hcl     # Version pinning and OCI labels
    tests.yaml          # Container structure tests
```

Published to:
- `ghcr.io/shockstruck/kronos-server:184.0`
- `ghcr.io/shockstruck/kronos-update-server:184.0`

## Build Requirements

- **Build stage**: `eclipse-temurin:21.0.10_7-jdk-alpine` (Gradle compilation)
- **Runtime stage**: `alpine:3.23` with `openjdk21-jre-headless`
- **Init system**: tini (Alpine native)
- **Build command**: `./gradlew :kronos-server:distZip` / `./gradlew :kronos-update-server:distZip`
- **Source/target compatibility**: Java 21, Kotlin JVM 21
- **Gradle wrapper**: Included in repo, build stage is self-contained

## Runtime Dependencies

### Database (MySQL/MariaDB)

- The `gamedb.sql` schema defines 30+ tables using InnoDB with UTF-8 charset.
- In **DEV** mode (`world_stage=DEV`), the database is optional. Player saving uses local files.
- In **LIVE** mode (`login_set=live`), a MySQL/MariaDB instance is required.
- Run the database as a sidecar container or connect to an external instance.

### Game Cache

- Binary `.dat2` and `.idx` files located in `Cache/` directory.
- These files are large (potentially 1GB+). Do **not** bake them into the Docker image.
- Mount as a volume at `/cache`.

### Player Save Data

- In DEV mode, player saves are stored as local files under the `Data/` directory.
- Requires a persistent volume mounted at `/data`.

## Configuration

The server reads `server.properties` at startup. Mount your config file to `/config/server.properties`.

Example values from `server.example.properties`:

```properties
cache_path=../Cache
data_path=Data
world_id=1
world_name=World 1
world_stage=DEV
world_type=PVP
world_flag=CANADA
world_settings=MEMBERS
world_address=0.0.0.0:13302
login_set=live
halloween=false
christmas=false
database_host=<host>
database_user=<user>
database_password=<password>
```

## Volume Mounts

| Mount Point | Purpose | Mode |
|---|---|---|
| `/config` | Server properties and configuration | read-only |
| `/data` | Player save data | read-write |
| `/cache` | Game cache files (`.dat2`, `.idx`) | read-only |

## Container Security

- Non-root execution: runs as `kronos` user (UID 1001)
- Multi-stage build: JDK build tools excluded from runtime image
- Alpine base: minimal attack surface
- tini init: proper PID 1 signal handling
- Health check: process-based liveness check
- OCI labels: full metadata for registry compliance

## Deployment

### Quick Start (DEV mode, no database)

```bash
docker run -d \
  --name kronos-server \
  -p 13302:13302 \
  -v ./cache:/cache:ro \
  -v ./data:/data \
  -v ./config:/config:ro \
  ghcr.io/shockstruck/kronos-server:184.0
```

### With Update Server

```bash
docker run -d \
  --name kronos-update-server \
  -p 7304:7304 \
  -v ./cache:/cache:ro \
  -v ./config:/config:ro \
  ghcr.io/shockstruck/kronos-update-server:184.0
```

## Feasibility Summary

| Factor | Rating | Detail |
|---|---|---|
| Build complexity | Medium | Multi-module Gradle + Kotlin + Lombok, but standard JVM build |
| Runtime complexity | Low | Single JVM process, well-suited for containers |
| Statefulness | Medium | Cache files + player saves need volumes |
| Network | Simple | Single TCP port per service |
| Database | Optional | Not needed in DEV mode |
| Multi-arch | Easy | JVM runs on any arch with matching JRE |
