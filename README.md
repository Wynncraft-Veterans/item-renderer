# item-renderer

Patch overlay over [Wynn-Tools/wynn.tools](https://github.com/wynn-tools/wynn.tools)
that exposes a server-side **PUA → PNG** route, so temporary-server can
render Wynncraft item-encoded chat lines into Discord image attachments.

Internal sidecar; not exposed publicly. The only consumer is
[`../temporary-server`](../temporary-server/)'s Discord bridge, which
hits `http://item-renderer:3000/api/og/pua/{encoded}` over the `verify`
Docker network.

## What this repo holds

```
patches/                          The entire functional delta.
docker/Dockerfile                 Clones upstream + applies patches + builds.
docker-compose.yml                Local dev convenience (port 3000).
.github/workflows/                CI: verify-patches (per-PR) + upstream-smoke (nightly).
```

No upstream wynn.tools source lives in this repo. The Docker build clones
it fresh from `wynn-tools/wynn.tools` at a configurable ref.

## How it works

1. The Dockerfile's `source` stage clones `wynn-tools/wynn.tools.git` at
   `WYNN_TOOLS_REF` (default = the pinned commit SHA in
   `docker/Dockerfile`).
2. `patches/*.patch` are applied with `git apply --check` first, then
   `git apply`. Application failure aborts the build with the offending
   patch named in the log.
3. The `builder` stage runs `pnpm install --frozen-lockfile && pnpm build`
   to produce the Nitro server output (`.output/`), which embeds the
   Takumi WASM OG renderer.
4. The final stage is a minimal `node:22-alpine` image with only the
   `.output/` tree.

The patch series adds:

- An `app/lib/wynntils/rolled-to-card.ts` mapper — turns a `parseIdString`
  decode result + base-item lookup into the structured props that
  `app/components/OgImage/ItemCard.takumi.vue` already accepts.
- An `app/pages/__og-image__/pua/[encoded].vue` page consumed by
  `nuxt-og-image`.
- A `server/api/og/pua/[encoded].get.ts` Nitro route that returns the PNG.

The upstream Hono `api/` backend is not used by the sidecar.

## Build / run locally

```bash
docker build -t item-renderer:local -f docker/Dockerfile .
docker run --rm -p 3000:3000 item-renderer:local
# then: GET http://localhost:3000/api/og/pua/<urlencoded-pua-string>
```

Or via compose:
```bash
docker compose up --build
```

## Tracking upstream

There is no manual sync. The Dockerfile clones at `WYNN_TOOLS_REF`;
bumping that var (in `docker/Dockerfile` or via build-arg) is the entire
upgrade process. CI's `upstream-smoke` workflow rebuilds **weekly**
(Sunday 06:00 UTC) against `WYNN_TOOLS_REF=main` and fails if patches
no longer apply or the build no longer compiles — that is the
early-warning signal.

A weekly cadence is intentional: the PUA renderer's failure mode is
graceful (temporary-server falls back to posting raw PUA text when a
render fails), so we don't need same-day notice of upstream breakage
the way auth-stack does. Trigger the workflow manually via
`workflow_dispatch` if you need to test sooner.

When `upstream-smoke` fails:

1. Read the job log to identify which patch broke (`git apply --check`
   names the file) or which build error appeared.
2. Locally regenerate the patch:
   ```bash
   git clone https://github.com/wynn-tools/wynn.tools.git /tmp/wt
   cd /tmp/wt
   git checkout <new-ref>
   git apply --reject /path/to/item-renderer/patches/000X-foo.patch
   # resolve *.rej hunks by hand, then:
   git add -u
   git diff --cached > /path/to/item-renderer/patches/000X-foo.patch
   ```
3. Bump `ARG WYNN_TOOLS_REF=` in `docker/Dockerfile` to the new SHA.
4. Open a PR; `verify-patches` will gate it. On merge the same workflow
   pushes `ghcr.io/wynncraft-veterans/item-renderer:<short-sha>` to GHCR.
5. Bump `IMAGE_TAG` in `vets-deploy/stacks/item-renderer/.env`, commit,
   push. On the VPS: `manage sync && manage update item-renderer`.

## Don't

- Don't add upstream wynn.tools source into this repo. The whole
  structural point is that they live only in upstream.
- Don't expose the sidecar publicly via Traefik — it's `verify`-only and
  has no auth.
- Don't bypass the patches by editing the running container. Patches are
  the source of truth.

## License

AGPLv3, inherited from upstream wynn.tools — see [LICENSE](LICENSE).
