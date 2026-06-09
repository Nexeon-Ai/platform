# Deploy Huly (Arabic + RTL) on Coolify

This folder deploys the Arabic/RTL build of Huly to a [Coolify](https://coolify.io)
server. The custom UI ships in the **front** image; all images come from this
fork's GitHub Container Registry (GHCR), built by the CI workflow.

## Overview

```
Internet ──TLS──> Coolify (Traefik) ──> nginx ──┬─> front (Arabic web client)
                                                ├─> account / workspace
                                                ├─> transactor (ws)
                                                ├─> collaborator (ws)
                                                ├─> fulltext, rekoni, stats
                                                └─> cockroach · redpanda · minio · elastic
```

## Step 1 — Publish the Arabic images to GHCR

The images aren't on Docker Hub; you build them once from your fork via GitHub Actions.

1. On `github.com/Nexeon-Ai/platform` → **Actions** tab → enable workflows if prompted.
2. Run **"Build & publish Arabic Huly images (GHCR)"** (`workflow_dispatch`), or just
   push to `feat/arabic-rtl`. It builds `front, transactor, account, workspace,
   collaborator, fulltext, stats, rekoni` and pushes them to
   `ghcr.io/nexeon-ai/huly-*` with tags `ar` and `ar-<sha>`.
3. After it finishes, open **github.com/orgs/Nexeon-Ai/packages** (or your user
   Packages) and either:
   - set each `huly-*` package to **Public** (simplest), **or**
   - keep them private and create a **PAT with `read:packages`** to use as a
     registry credential in Coolify (Step 3).

> The build is heavy (full monorepo). The workflow reuses the repo's
> `free-disk-space` action and is capped at 150 min; a normal run is well under that.

## Step 2 — Create the resource in Coolify

1. **New Resource → Docker Compose** (Git-based), point it at this repo/branch and
   set the **Base/Compose path** to `coolify/docker-compose.yml`.
2. **Environment Variables**: paste everything from [`.env.example`](./.env.example).
   At minimum set:
   - `HOST_ADDRESS` = your domain (no scheme), e.g. `huly.example.com`
   - `SECRET` = output of `openssl rand -hex 32`
   - `SECURE=s` (TLS), `HULY_IMAGE_REGISTRY`, `HULY_IMAGE_TAG=ar`
3. **Domain**: assign `https://<your-domain>` to the **`nginx`** service on **port 80**.
   Make sure `HOST_ADDRESS` matches that domain exactly. Point the domain's DNS
   A record at your Coolify server first.

## Step 3 — (If images are private) add a GHCR pull credential

Coolify → **Keys & Tokens / Registries** → add `ghcr.io` with your GitHub username
and the `read:packages` PAT, then attach it to the resource. Skip this if you made
the packages public.

## Step 4 — Deploy

Hit **Deploy**. First boot pulls the base images (cockroach, redpanda, minio,
elastic) and your `huly-*` images, then initialises the database and storage.
Give it a few minutes; watch logs for `account`, `transactor`, `front` going healthy.

## Step 5 — Use it

Open `https://<your-domain>`, **Sign Up**, create a workspace. The UI defaults to
Arabic (`DEFAULT_LANGUAGE=ar`) with full RTL; you can switch languages anytime from
the settings (gear) popup.

## Resources & notes

- **Minimum host**: ~4 GB RAM is tight (Elasticsearch alone wants ~1 GB); **8 GB+**
  recommended. CPU 2+ cores. Persistent volumes are declared for cockroach, minio,
  elasticsearch and redpanda — back these up.
- **Optional services** intentionally omitted for a lean install: full-text search
  works (elastic+fulltext+rekoni included), but love/calendar/gmail/telegram/github/
  print/sign and `hulykvs` are not wired. Add them later from the upstream
  `huly-selfhost` compose if needed.
- **Updating**: re-run the GHCR workflow (new `ar-<sha>` tag), then redeploy in
  Coolify. Pin `HULY_IMAGE_TAG` to a specific `ar-<sha>` for reproducible deploys.
- **MinIO** uses the default `minioadmin/minioadmin` credentials on the internal
  network only; change `STORAGE_CONFIG` + MinIO env if you expose it.
