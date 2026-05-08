# Docker Basic

This course covers Docker from first principles to production-ready workflows. You will learn how containers work, how to build and distribute images, manage data and networking, and orchestrate multi-container applications with Docker Compose.

## Curriculum

<!-- 4 sections · 22 themes -->

### Docker Fundamentals and Image Building

- [Containerization Concepts: Containers vs Virtual Machines](./docker-fundamentals-and-image-building/containerization-concepts.md)
- [Installing Docker and First Commands](./docker-fundamentals-and-image-building/installing-docker-first-commands.md)
- [Dockerfile Structure and Core Instructions](./docker-fundamentals-and-image-building/dockerfile-structure-instructions.md)
- [CMD vs ENTRYPOINT: Defining Container Behavior](./docker-fundamentals-and-image-building/cmd-vs-entrypoint.md)
- [docker build, Layer Caching and Cache Optimisation](./docker-fundamentals-and-image-building/docker-build-caching.md)
- [Multi-Stage Builds and Image Size Optimisation](./docker-fundamentals-and-image-building/multi-stage-builds.md)

### Running Containers and Image Distribution

- [docker run Options](./running-containers-and-image-distribution/docker-run-options.md)
- [Container Lifecycle: exec, logs, inspect, stats, top](./running-containers-and-image-distribution/container-lifecycle.md)
- [Cleanup: Removing Containers, Images, and Reclaiming Space](./running-containers-and-image-distribution/cleanup.md)
- [Image Distribution: Registry, Push, Pull, and Private Registries](./running-containers-and-image-distribution/image-distribution.md)
- [Docker Security Basics: Non-Root Users and Minimal Base Images](./running-containers-and-image-distribution/security-basics.md)

### Volumes and Data Persistence

- [Ephemeral Containers and the Writable Layer](./volumes-and-data-persistence/ephemeral-containers.md)
- [Named Volumes](./volumes-and-data-persistence/named-volumes.md)
- [Bind Mounts and tmpfs](./volumes-and-data-persistence/bind-mounts-tmpfs.md)
- [--mount vs -v Syntax](./volumes-and-data-persistence/mount-syntax.md)
- [Volume Backup, Restore, and Sharing Between Containers](./volumes-and-data-persistence/volume-backup-sharing.md)

### Networking and Docker Compose

- [Docker Network Types: bridge, host, none](./networking-and-docker-compose/network-types.md)
- [Network Commands and Port Publishing](./networking-and-docker-compose/network-commands-ports.md)
- [Container-to-Container Communication](./networking-and-docker-compose/container-communication.md)
- [What Docker Compose Solves and docker-compose.yml Structure](./networking-and-docker-compose/compose-structure.md)
- [Compose Services: Configuration Reference](./networking-and-docker-compose/compose-services.md)
- [Docker Compose Commands, Dependencies, and Healthchecks](./networking-and-docker-compose/compose-commands.md)
