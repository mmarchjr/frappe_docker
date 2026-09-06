---
title: Dokploy Deployment
---

# Dokploy Deployment (Frappe v16 + ERPNext + HRMS + BuildSuite Core)

This guide keeps the upstream `frappe_docker` architecture and uses the existing custom image + compose override flow.

## 1) Build and push the custom image

Use the repository `apps.json` (already configured for `erpnext`, `hrms`, `buildsuite_core`) and build with BuildKit secret:

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --build-arg=CACHE_BUST="$(sha256sum apps.json | awk '{print $1}')" \
  --secret=id=apps_json,src=apps.json \
  --tag=ghcr.io/<your-org>/frappe-buildsuite:v16 \
  --file=images/layered/Containerfile .
```

Push it to your registry:

```bash
docker push ghcr.io/<your-org>/frappe-buildsuite:v16
```

## 2) Prepare environment variables

In Dokploy, define these environment variables for the app:

- `ERPNEXT_VERSION=v16.34.1`
- `CUSTOM_IMAGE=ghcr.io/<your-org>/frappe-buildsuite`
- `CUSTOM_TAG=v16`
- `PULL_POLICY=always`
- `DB_PASSWORD=<strong-db-password>`
- `SITE_NAME=<your-domain>`
- `ADMIN_PASSWORD=<strong-admin-password>`
- `FRAPPE_SITE_NAME_HEADER=<your-domain>`

Optional:

- `DB_ROOT_USER=root`
- `MIGRATE_SITES=true`
- `CLIENT_MAX_BODY_SIZE=50m`
- `PROXY_READ_TIMEOUT=120`

## 3) Assemble Dokploy Compose file

Generate one final compose file from upstream base + overrides:

```bash
docker compose \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.create-site.yaml \
  -f overrides/compose.migrator.yaml \
  -f overrides/compose.noproxy.yaml \
  config > compose.dokploy.yaml
```

Commit `compose.dokploy.yaml` to your repo and select it in Dokploy.

## 4) Configure Dokploy application

1. Create a Docker Compose application in Dokploy from this repository.
2. Set compose file path to `compose.dokploy.yaml`.
3. Add registry credentials in Dokploy if your image is private.
4. Add the environment variables from step 2.
5. Configure Dokploy domain to route traffic to the `frontend` service on port `8080`.

## 5) Deploy

Start deployment from Dokploy.

What happens on first deploy:

1. `configurator` writes bench config.
2. `create-site` creates the site if missing.
3. Apps are installed in order: `erpnext` → `hrms` → `buildsuite_core`.
4. `migrator` runs `bench --site all migrate`.
5. Regular services run: backend, frontend, workers, scheduler, websocket.

## 6) Verify

Check logs in Dokploy for `create-site` and `migrator`, then validate apps:

```bash
docker compose -f compose.dokploy.yaml exec backend bench --site <your-domain> list-apps
```

Expected apps:

- `frappe`
- `erpnext`
- `hrms`
- `buildsuite_core`
