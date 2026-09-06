---
title: Dokploy Deployment
---

# Dokploy Deployment (Frappe v16 + ERPNext + HRMS + BuildSuite Core + Employee Self Service)

This guide keeps the upstream `frappe_docker` architecture and uses a Dokploy-ready `docker-compose.yml` that builds the custom layered image directly.

## 1) Use the provided Dokploy files

This repository now includes:

- `docker-compose.yml` (Dokploy entry file, includes production services + MariaDB + Redis + create-site + migrator)
- `.env.dokploy.example` (complete environment template)
- `apps.json` (custom app list for build secret: `erpnext`, `hrms`, `buildsuite_core`, `employee_self_service`)

## 2) Prepare environment variables

Copy `.env.dokploy.example` to `.env` and set real values:

```bash
cp .env.dokploy.example .env
```

At minimum, update:

- `SITE_NAME`
- `DB_PASSWORD`
- `ADMIN_PASSWORD`
- `FRAPPE_SITE_NAME_HEADER`
- `CUSTOM_IMAGE` (optional custom image name/tag for your Dokploy project)

Then put the same key/value pairs into Dokploy app environment variables.

## 3) Configure Dokploy application

1. Create a Docker Compose application in Dokploy from this repository.
2. Set compose file path to `docker-compose.yml`.
3. Add environment variables from step 2.
4. Enable build/deploy so Dokploy builds services from the compose `build` definition.
5. Configure Dokploy domain to route traffic to the `frontend` service on port `8080`.

## 4) Deploy

Start deployment from Dokploy.

What happens on first deploy:

1. `configurator` writes bench config.
2. `create-site` creates the site if missing.
3. Apps are installed in order: `erpnext` → `hrms` → `buildsuite_core` → `employee_self_service`.
4. `migrator` runs `bench --site all migrate`.
5. Regular services run: backend, frontend, workers, scheduler, websocket.

## 5) Verify

Check logs in Dokploy for `create-site` and `migrator`, then validate apps:

```bash
docker compose -f docker-compose.yml exec backend bench --site <your-domain> list-apps
```

Expected apps:

- `frappe`
- `erpnext`
- `hrms`
- `buildsuite_core`
- `employee_self_service`
