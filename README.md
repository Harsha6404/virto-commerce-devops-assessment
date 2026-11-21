# Virto Commerce Docker Compose Assessment

This repository contains the Docker Compose configuration and Docker build definitions for a local Virto Commerce assessment deployment. It describes a Docker-based application stack; it does not provision or claim to represent an AWS deployment.

## Architecture

- **Platform**: Virto Commerce Platform 3.1076.13, exposed on the host at `127.0.0.1:8090`.
- **Storefront**: official Vue B2B frontend artifact 2.59.1, served by Nginx on `127.0.0.1:80`.
- **Dependencies**: PostgreSQL, Redis, and Elasticsearch on the private Compose network. Kibana is disabled by the assessment override initially.
- The frontend Nginx configuration proxies API requests to the `vc-platform-web` Compose service. Containers communicate through the internal `virto` network and Compose DNS; PostgreSQL, Redis, and Elasticsearch are not published to host ports in the assessment override.
- Named volumes retain PostgreSQL, Redis, Elasticsearch, CMS content, and module data across container recreation. Do not remove volumes if you need to preserve application data.

## Versions

- Virto Commerce Platform: **3.1076.13**
- Vue B2B frontend: **2.59.1**
- .NET SDK used to prepare backend modules: **10.0.115**
- Virto build tool used: **VirtoCommerce.GlobalTool 3.1001.0**

## Prerequisites

- Linux x86_64 host with Docker Engine and Docker Compose v2 or later.
- .NET SDK 10.0.115 and the global `VirtoCommerce.GlobalTool` 3.1001.0 (`vc-build`) for backend artifact preparation.
- `curl` and `unzip` for downloading and extracting the frontend release.
- At least 2 CPU cores, 7.6 GB RAM, and sufficient free disk for the application images, artifacts, and persistent data. Elasticsearch is constrained to 1 GB by default in this assessment configuration.

## Prepare build artifacts

Run commands from the repository root. The generated directories below are intentionally excluded from Git.

### Backend

The pinned package manifest is `backend/stable-packages.json`. Install its modules into the publish tree with the pinned build tool:

```bash
vc-build install \
  --package-manifest-path "$PWD/backend/stable-packages.json" \
  --probing-path "$PWD/backend/publish/app_data/modules" \
  --discovery-path "$PWD/backend/publish/modules" \
  --root "$PWD/backend/publish" \
  --skip-dependency-solving
```

The backend Dockerfile expects `backend/publish/` and `backend/wait-for-it.sh` in its build context. The manifest records the pinned Platform package set; verify its Platform version before using it.

### Frontend

Download the official pinned release and extract it so `frontend/artifact/default/index.html` exists:

```bash
curl -fL \
  https://github.com/VirtoCommerce/vc-frontend/releases/download/2.59.1/vc-frontend-2.59.1.zip \
  -o frontend/frontend.zip
mkdir -p frontend/artifact
unzip -q frontend/frontend.zip -d frontend/artifact
```

The frontend Dockerfile copies `frontend/artifact/default/` into the Nginx document root. Confirm the extracted directory layout before building.

## Configure environment

Copy `.env.example` to `.env` and replace every `replace-with-...` value with a different randomly generated secret. For example, generate a value with `openssl rand -hex 32`; do not reuse credentials. Restrict the file to the owner (`chmod 600 .env`). Never commit `.env` or paste its contents into logs or support requests.

The Compose files require `DB_PASSWORD`, `REDIS_PASSWORD`, and `ELASTIC_PASSWORD` and fail configuration if they are unset. `KIBANA_PASSWORD` is only used if the optional Kibana service is enabled. The supplied port and memory settings are suitable starting values for a small assessment host; adjust only after reviewing available resources.

## Build and run

Build the images from the repository root after preparing both artifacts:

```bash
docker build -t vc-platform:local-latest -f backend/Dockerfile backend/
docker build -t vc-frontend:local-latest -f frontend/Dockerfile frontend/
```

Validate the merged configuration, then start the stack:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.postgres.yml \
  -f docker-compose.assessment.yml config

docker compose \
  -f docker-compose.yml \
  -f docker-compose.postgres.yml \
  -f docker-compose.assessment.yml up -d
```

The assessment override binds only the frontend and Platform to loopback. It disables Kibana initially and prevents host port publication for PostgreSQL, Redis, and Elasticsearch. Do not add public firewall access for these services.

## Access locally

- Storefront: `http://127.0.0.1/`
- Virto Manager / Platform: `http://127.0.0.1:8090/`

On a remote Linux host, create an SSH tunnel from your workstation, for example:

```bash
ssh -L 8080:127.0.0.1:80 -L 8090:127.0.0.1:8090 ubuntu@YOUR_HOST
```

Then open `http://127.0.0.1:8080/` for the storefront and `http://127.0.0.1:8090/` for Manager from the workstation. This forwards local traffic through SSH; it does not publish the application ports publicly.

## Security and operational notes

- This is an assessment/local Docker Compose setup, not a production-hardened deployment.
- Keep `.env` private and use unique credentials. Do not rely on defaults or commit generated secrets.
- The Compose network is private to the Docker host; only loopback-bound frontend and Platform ports are published by the assessment override.
- The included base image tags follow upstream tags and may change over time. For long-lived reproducible builds, pin image digests and review updates.
- Persistent volumes retain state. Avoid `docker compose down -v` and Docker prune commands when data must be preserved.
- No AWS resources, security groups, or cloud infrastructure are created by these files.
