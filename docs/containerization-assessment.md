# Kronos-184: Containerization Assessment

## Project Overview

Kronos is an OSRS (Old School RuneScape) private game server. It is a Java 21 multi-module Gradle project using Kotlin 2.1.21, Lombok, and Netty for networking.

## Components

| Component | Port | Containerize? | Notes |
|---|---|---|---|
| `kronos-server` | 13302 | **Yes** | Main game server, primary target |
| `kronos-update-server` | 7304 | **Yes** | Cache update server, separate container |
| `kronos-webhooks` | - | **Yes** | Discord integration, separate container |
| `runelite` (client) | - | **No** | Desktop GUI app, not suitable |
| `kronos-launcher` | - | **No** | Client launcher, not a server |
| `Kronos-web` (PHP) | - | **Maybe** | Simple PHP scripts, could use nginx+php-fpm |

## Build Requirements

- **Base image**: `eclipse-temurin:21-jre-alpine` (runtime)
- **Build stage**: JDK 21 + Gradle 8.14 (multi-stage build)
- **Build command**: `./gradlew :kronos-server:distZip`
- **Source/target compatibility**: Java 21, Kotlin JVM 21
- **Gradle wrapper**: Included in repo (`gradlew`), build stage is self-contained

## Runtime Dependencies

### Database (MySQL/MariaDB)

- The `gamedb.sql` schema defines 30+ tables using InnoDB with UTF-8 charset.
- In **DEV** mode (`world_stage=DEV`), the database is optional. Player saving uses local files.
- In **LIVE** mode (`login_set=live`), a MySQL/MariaDB instance is required.
- Run the database as a sidecar container or connect to an external instance.

### Game Cache

- Binary `.dat2` and `.idx` files located in `Cache/` directory.
- These files are large (potentially 1GB+). Do **not** bake them into the Docker image.
- Mount as a volume or use an init container to populate them.

### Player Save Data

- In DEV mode, player saves are stored as local files under the `Data/` directory.
- Requires a persistent volume to survive container restarts.

## Configuration

The server reads `server.properties` at startup. Example values from `server.example.properties`:

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

For containerization, these values should be configurable via:

- Environment variables injected through an entrypoint script
- A mounted config file at a known path

## Challenges

### 1. Relative Path References

The config uses `cache_path=../Cache` (parent directory). The container working directory and file layout must match this expectation.

**Solution**: Structure the container as:

```
/app/
  Cache/          <- mounted volume
  server/
    bin/           <- gradle distZip output
    lib/           <- gradle distZip output
    Data/          <- mounted volume
    server.properties
```

Set the working directory to `/app/server/` so `../Cache` resolves to `/app/Cache/`.

### 2. Large Binary Cache Files

The `Cache/` directory contains binary game assets that should not be included in the Docker image layer.

**Solution**: Use a Docker volume mount. Populate the cache before first run, either manually or via an init container.

### 3. State Persistence

Player data in `Data/` must persist across container restarts.

**Solution**: Mount `Data/` as a persistent volume.

### 4. Database Connectivity

In LIVE mode, the server needs MySQL access.

**Solution**: Use a `docker-compose.yml` or Kubernetes deployment with a MySQL sidecar. Pass connection details via environment variables.

## Recommended Architecture

```
+-----------------------+     +------------------------+
|  kronos-server        |---->|  MySQL/MariaDB          |
|  (JDK 21 Alpine)      |     |  (optional, LIVE only)  |
|  Port: 13302          |     +------------------------+
+-----------------------+
        |
   Volumes:
     /app/Cache    (game cache, read-only)
     /app/server/Data  (player saves, read-write)
     /app/server/server.properties (config, read-only)

+-----------------------+
|  kronos-update-server |
|  Port: 7304           |
+-----------------------+
        |
   Volumes:
     /app/Cache    (shared with game server)
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

## Next Steps

1. Create a multi-stage `Dockerfile` for `kronos-server`
2. Create a `docker-bake.hcl` build definition
3. Add a `docker-compose.yml` for local development (server + MySQL)
4. Test the build in DEV mode (no database required)
5. Validate LIVE mode with MySQL connectivity
