# Docker: Build and Run an Application Container

[Back to the learning package](README.md) · [Kubernetes](kubernetes.md) · [Learning path](learning-path.md)

## The problem Docker solves

An application that works on a developer's laptop may fail elsewhere because the runtime, system libraries, application dependencies, filesystem layout, or startup command differs. Docker provides a repeatable way to build an image containing the application and its user-space runtime dependencies, then run that image in a container.

A Linux container is an isolated process environment that shares the host kernel; it is not a full virtual machine with its own guest kernel. Docker Desktop on macOS runs Linux containers inside a lightweight Linux VM because macOS does not provide the Linux kernel interfaces those containers need.

Containerization reduces environment drift, but it does not make software fully portable by magic. CPU architecture, kernel capabilities, external services, secrets, network policy, and persistent data still matter.

## The build-to-run model

```text
source + Dockerfile + build context
        |
        v
image (layers, tagged or addressed by digest)
        |
        +--> registry (shareable image artifact)
        |
        v
container (running process with isolated filesystem/network/resources)
```

| Term | Meaning | Practical note |
| --- | --- | --- |
| Dockerfile | Ordered build instructions | `FROM`, `COPY`, `RUN`, `USER`, `CMD`, and other instructions define the image |
| Build context | Files made available to the builder | A `.dockerignore` file prevents irrelevant or sensitive files from entering the context |
| Image | Versioned, layered filesystem and metadata used to create containers | A tag is a human-readable reference and can be moved; a digest identifies exact content |
| Container | A running or stopped instance created from an image | Its writable layer is disposable; do not treat it as durable storage |
| Registry | Service that stores and distributes images | Nodes need registry access or a locally loaded image |
| Volume | Data mounted separately from the container's writable layer | Use volumes for Docker-managed persistent data; use bind mounts for host-visible development files |
| Network | Connectivity between containers and other endpoints | On a user-defined network, containers can use service/container names rather than hard-coded IPs |

### Build a tiny learning image

Assume `app.py` serves an HTTP endpoint on `0.0.0.0:8000`. A simple Dockerfile is:

```dockerfile
FROM python:3.13-slim

WORKDIR /app
COPY --chown=10001:10001 app.py .
USER 10001:10001

EXPOSE 8000
CMD ["python", "app.py"]
```

Build and run it locally:

```sh
docker build --pull -t sample-api:dev .
docker run --rm --name sample-api -p 8080:8000 sample-api:dev
```

In another terminal, try `curl http://localhost:8080/`. The `-p` flag publishes host port `8080` to container port `8000`. `EXPOSE` documents the container's listening port; it does not publish that port to the host by itself.

This minimal image is a learning example, not a universal production Dockerfile. Production images should use the application's locked dependencies, a reviewed base image, an appropriate build/runtime separation, a known non-root identity, and a release-specific tag or digest.

### Build-context hygiene

Add a `.dockerignore` file appropriate to the repository. For example:

```text
.git
.venv
__pycache__/
*.pyc
.env
```

Review exclusions before copying a repository into a build context. Credentials, local environment files, caches, virtual environments, and large test artifacts should not accidentally be sent to a builder or copied into an image.

## Image quality: reliable, fast, and safer to ship

1. **Copy dependency manifests before frequently changing source.** This allows Docker to reuse cached dependency-installation layers when application code changes.
2. **Use multi-stage builds when the build toolchain is not needed at runtime.** Copy only the final binary or runtime artifacts into the final stage.
3. **Choose a maintained base image with only the required runtime.** Smaller images can reduce transfer time and the number of included components, but verify compatibility and patching practices.
4. **Use `.dockerignore` and explicit `COPY` instructions.** This keeps the build context controlled and reduces accidental inclusion.
5. **Run as a non-root user when feasible.** Also minimize Linux capabilities and writable paths appropriate to the application.
6. **Keep containers replaceable.** Persist required data outside the container layer; handle configuration at runtime.
7. **Never pass build credentials through `ARG` or `ENV`.** Use BuildKit secret or SSH mounts for secrets needed only during a build. Runtime credentials belong in a suitable secret-delivery system.
8. **Rebuild and rescan images.** Immutable image contents do not receive security fixes automatically.
9. **Tag deliberately.** A mutable tag such as `latest` is convenient for a tutorial but risky for reproducible deployments. Use a unique release tag and, where needed, pin the image digest.
10. **Check platform compatibility.** An Apple Silicon build may produce an ARM64 image while a deployment target expects AMD64; use a deliberate multi-platform build and test each target when necessary.

The official [build best practices](https://docs.docker.com/build/building/best-practices/), [multi-stage build guide](https://docs.docker.com/build/building/multi-stage/), and [build-secret guide](https://docs.docker.com/build/building/secrets/) explain these choices in more detail.

## Configuration, storage, and networking

### Configuration and secrets

Configuration should not be baked into the image when it varies by environment. Supply non-secret configuration at runtime. Do not put credentials in the Dockerfile, image layer, build arguments, committed `.env` files, or command history. A local `.env` file can be convenient for Compose, but protect and exclude it from version control and build contexts; it is not by itself a production secret manager.

### Container storage

Deleting a container deletes its writable layer. Use a named volume when container data must survive replacement. Use a bind mount when a developer needs to edit or inspect host files. Back up important data separately: a Docker volume is persistence, not a complete backup or recovery plan.

For example, `docker compose down` normally preserves named volumes; adding `--volumes` removes them and their data. Use that option only when you intend to reset the local exercise.

### Container networking

Containers on the same user-defined network can communicate using their service names and container ports. `localhost` inside a container refers to that container, not another service and not automatically the developer's host. Use a published host port for host-to-container access.

In Compose, a service called `db` is normally reached by another service as `db:<container-port>`. In Kubernetes, use a Kubernetes Service name rather than a Pod IP; see [Kubernetes networking](kubernetes.md#networking-and-exposure).

## Docker Compose: local multi-service applications

Compose describes services, networks, and volumes together and provides a convenient lifecycle for local development, tests, and CI. It can also be used in some single-host deployments; it is not limited to development. It is not a substitute for Kubernetes when the requirement is multi-node scheduling and cluster-level reconciliation.

For a local API and database, define an `api` service and a `db` service, place them on a Compose network, configure the API to connect to host `db`, and mount a named volume to the database's data directory. Use a health check if the API must wait until the database can accept connections; simple start order alone does not mean a dependency is ready.

Common commands:

```sh
docker compose config
docker compose up --build
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose down
```

Use the official [Compose guide](https://docs.docker.com/compose/) and [production guidance](https://docs.docker.com/compose/how-tos/production/) to understand layered configuration and the limits of a single-server setup.

## Useful commands and first-line debugging

| Question | Command |
| --- | --- |
| Did the image build? | `docker image ls` |
| What process is running? | `docker ps` and `docker inspect sample-api` |
| Why did it exit? | `docker logs sample-api` |
| Is the process using resources? | `docker stats sample-api` |
| Can I inspect the container environment? | `docker exec -it sample-api sh` (if the image includes a shell) |
| Does host port mapping work? | `docker port sample-api` and `curl http://localhost:8080/` |
| Did Compose parse the configuration I intended? | `docker compose config` |

If a container exits, check its exit code, logs, command/entrypoint, environment, filesystem paths, and architecture before changing Kubernetes settings. A Kubernetes Deployment cannot repair an application that fails identically when run locally.

## When Docker is not enough

- Docker by itself runs a container on one Docker host; it does not provide a cluster scheduler for many machines.
- Compose describes multiple services but does not give the same declarative cluster controller and scheduling model as Kubernetes.
- A container image does not include durable external data, cloud infrastructure, runtime secrets, or all host-kernel behavior.
- An image build is not a CI/CD policy, vulnerability review, backup plan, or production health check.

Continue to [Kubernetes](kubernetes.md) when you need cluster-level workload scheduling and lifecycle management, or use the [capstone lab](learning-path.md) to compare Compose and Kubernetes on the same service.
