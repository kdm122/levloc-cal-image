# levloc-cal-image

Builds a Docker image of [`calcom/cal.diy`](https://github.com/calcom/cal.diy)
(the MIT community fork of Cal.com) from a **pinned upstream commit** and pushes
it to **GHCR**, because upstream publishes no official `cal.diy` image.

The image runs on Railway in the `levloc-cal` project as `https://cal.levloc.com`.

- **Image:** `ghcr.io/kdm122/cal.diy:<short-sha>`
- **Current pinned SHA:** `6bc45298226f96ff79e0c070c8b2ce39727e8477` (upstream `main`, 2026-09-14)
- **Nothing secret is in this repo or the image.** Real secrets live only in Railway.

## One-time setup

1. Create this repo on your personal account as **`kdm122/levloc-cal-image`** and
   keep it **Public**. No org, no Team plan, no paid runner: public repos get free
   `ubuntu-latest` 4-core / 16 GB runners with unlimited minutes, which is enough
   for this build. (Private repos only get 2-core / 7 GB runners, which OOM here.)
2. Push `build.yml` + this README to the repo's default branch.
3. Run the workflow once (below). After the first push, go to your
   **Packages → cal.diy → Package settings** and set visibility to **Public**
   (so Railway pulls with no credentials and image-update detection works).

## Build / release

**Actions → build-cal-diy-image → Run workflow**, enter the `upstream_sha`
(defaults to the pinned SHA), leave *push* checked. ~25–40 min, **free**
(public-repo runners bill no minutes).

The run summary prints the exact tag to deploy on Railway.

## Upgrade runbook

1. Pick a new upstream SHA from https://github.com/calcom/cal.diy/commits/main .
   Skim commits touching `packages/prisma/migrations` since the current SHA.
2. Run the workflow with that SHA.
3. In Railway (`levloc-cal` → Postgres) take a **manual volume backup**.
4. In Railway (`web` service → Settings → Source) change the image tag to the new
   `:<short-sha>`. Deploy. Watch logs for `prisma migrate deploy` and the
   `/api/version` healthcheck passing (migrations run automatically in `start.sh`).
5. Smoke test (login, make a booking, cancel). Optionally `POST /api/cron/syncAppMeta`.
6. Update the "Current pinned SHA" line above and commit.

**Rollback:** re-point `web` at the previous `:<short-sha>` and redeploy. If the
new release added non-additive migrations, restore the pre-upgrade Postgres
volume backup first, then redeploy the old tag.

## How the build is customized

Upstream's `Dockerfile` only accepts a fixed set of build args, and Next.js
inlines `NEXT_PUBLIC_*` values **at build time**. `build.yml` therefore patches
the Dockerfile to add `ARG`/`ENV` lines for the branding + signup flags before
the web build, then passes them (plus terms/privacy URLs, telemetry off, license
consent, and the baked `NEXT_PUBLIC_WEBAPP_URL=https://cal.levloc.com`) as build
args. The patch step **fails loudly** if the upstream Dockerfile layout drifts,
so a bad SHA bump can't silently drop the branding.

A monthly scheduled run does a **build-only** canary (no push) to catch upstream
breakage early.
