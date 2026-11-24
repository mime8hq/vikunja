# Deploying Vikunja to Cloudflare

This fork runs at <https://vikunja.mime8.dev> as a test app for mime8. One Worker serves the frontend (this repo's `frontend/dist`) as static files and sends `/api/*` to a Cloudflare Container running the Vikunja API.

- **API image:** Vikunja's own image (`vikunja/vikunja:unstable`, pinned by digest in `Dockerfile`, built from the same upstream commit as this fork's `main`) plus [Litestream](https://litestream.io).
- **Data:** a container's disk is wiped when it sleeps. The SQLite database is restored from R2 on start and replicated back to R2 while Vikunja runs (`litestream.yml`, `start.sh`); attachments are stored in R2 directly. Bucket: `vikunja-staging`.
- **One instance only** (`max_instances: 1`): Litestream supports a single writer.
- **Settings** are in the `Dockerfile` (`VIKUNJA_*`). A running container keeps the environment it first started with, even across a new image's rollout, so a change (and a changed secret) takes effect when the container next starts, after it has slept (`sleepAfter`).
- **Registration** is off; set `VIKUNJA_SERVICE_ENABLEREGISTRATION=true` only while creating accounts.

## Secrets

Set once per Worker (`wrangler secret put <NAME>`, or the dashboard: Workers & Pages → vikunja-staging → Settings → Variables and Secrets):

- `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`: an R2 account API token with Object Read & Write on `vikunja-staging` only.
- `VIKUNJA_SECRET`: signs sessions; changing it logs everyone out. `openssl rand -hex 32 | wrangler secret put VIKUNJA_SECRET`.

## Deploy

```sh
(cd ../../frontend && corepack pnpm install --frozen-lockfile && corepack pnpm run build)
npm install
npx wrangler deploy
```

After syncing the fork with upstream, re-pin the API image to the matching `unstable` digest (`docker buildx imagetools inspect vikunja/vikunja:unstable`) and check that `/api/v1/info` reports the fork's commit.

PRs on this fork get a `mime8` check: `.github/workflows/mime8.yml` uploads the PR's base and head builds with `@mime8/cli`.
