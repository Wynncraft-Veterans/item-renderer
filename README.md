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
3. The `builder` stage runs `pnpm install` (pnpm pinned in the
   Dockerfile), type-checks every TypeScript file the patches add or
   change against that upstream ref, then `pnpm build` to produce the
   Nitro server output (`.output/`), which embeds the Takumi WASM OG
   renderer. `nuxt build` alone doesn't type-check, so without step 3 an
   upstream rename our code relies on builds green and ships broken
   cards. Upstream isn't type-clean, so only errors in our files count.
4. The final stage is a minimal `node:22-alpine` image with only the
   `.output/` tree.

The patch series adds, following the
[Arethusa spec](https://gist.github.com/wenweia/83721cfeeafb92d0cb85466bde7e8738)
for the item encoding:

- `app/lib/wynntils/rolled-to-card.ts` (0001) — turns a `parseIdString`
  decode result + base item into the props
  `app/components/OgImage/ItemCard.takumi.vue` already accepts. Current
  strings carry the value the tooltip shows; legacy ones carry an internal
  roll that gets rebuilt; spell costs travel sign-flipped in both.
- `app/lib/wynntils/pua-item.ts` (0001) — finds the base item: by in-game
  display name, `Ascended <name>` / `Masterwork <name>` for ascended
  mythics, generic names for crafted items and mounts, and a reload of the
  item data when the CDN publishes a newer snapshot.
- `server/plugins/pua-warmup.ts` (0001) — loads that item data and the
  stat ids at startup, so the first renders after a restart don't spend
  the bridge's timeout on it.
- `server/api/_og/pua/[encoded].get.ts` (0001, 0003) — the Nitro route
  that returns the PNG, or a 404 whose `Unknown item: <name>` reason the
  bridge uses in its text fallback.
- `app/lib/wynntils/best-effort-name.ts` (0003) — a lax name walker for
  strings the strict decoder rejects.
- `pnpm-workspace.yaml` policy knobs (0002) — see the Dockerfile.

The upstream Hono `api/` backend is not used by the sidecar.

## Build / run locally

```bash
docker build -t item-renderer:local -f docker/Dockerfile .
docker run --rm -p 3000:3000 item-renderer:local
# then: GET http://localhost:3000/api/_og/pua/<urlencoded-pua-string>
```

Or via compose:
```bash
docker compose up --build
```

## Tracking upstream

Automatic. The `upstream-track` workflow runs daily at 06:00 UTC (and
on every main push), queries `https://api.github.com/repos/wynn-tools/wynn.tools/commits/main`
for the latest main-branch SHA, and rebuilds against that SHA. On a
green build, both `:<short-sha>` and `:rolling` are pushed. On a red
build, **nothing** is pushed — `:rolling` keeps its previous digest,
which is the implicit last-known-good state. GHCR is the state store;
there is no separate tracking file.

**Smart-skip on schedule:** wynn.tools main is high-velocity, but most
commits touch pages and components we don't render from (stock/build
editors, world-events pages, etc.). The daily cron reads the previous
`:rolling` image's `org.wynnvets.upstream-ref` label and calls the
GitHub compare API. If nothing changed under our relevant surface —
`app/lib/**`, `app/components/OgImage/**`, `server/api/_og/**`, or the
root build config — the build is skipped entirely. Push, PR, and
`workflow_dispatch` runs always build. If a new import surface enters
our patches, extend the `patterns=(…)` allowlist in
`.github/workflows/upstream-track.yml` in the same PR.

**Keeping the schedule alive:** GitHub disables a public repo's
scheduled workflows after 60 days without repository activity, and a
disabled workflow doesn't run on push or pull requests either. That
happened on 2026-09-08 and went unnoticed until 2026-10-20, so
upstream's Arethusa 3.0 decoder (2026-09-20) never reached production
and nearly every share fell through to the bridge's Wynnpool fallback.
The workflow's `keepalive` job now re-enables it on every scheduled
run. If it is ever disabled anyway, the Actions tab says so on the
`upstream-track` page: click **Enable workflow** (or
`gh workflow enable upstream-track -R Wynncraft-Veterans/item-renderer`),
then run it once by hand.

When a scheduled build goes red:
- Inspect the failed run. If `git apply` failed or `pnpm build` failed,
  regenerate the affected patch — see the "regenerate patches" recipe
  below. If the type check failed, the log names the patched file and
  the error: upstream changed something that patch relies on, so update
  it the same way.
- Production keeps running on the previous `:rolling` digest in the
  meantime; there's no urgent rollback needed. The temporary-server
  bridge falls back to a Wynnpool-decoded text card, then to raw PUA
  text, if any render fails, so end-users see slightly-worse output
  rather than nothing.

When you need to manually pin (e.g. `:rolling` shipped a runtime-bad
image before it was caught):
```
manage pin item-renderer 1a2b3c4      # pin to a specific known-good short SHA
manage pin item-renderer rolling      # resume auto-tracking
```
`manage pin` lives in vets-deploy's `scripts/manage.sh`. It writes
`IMAGE_TAG` into the stack's `.env` and runs `docker compose pull && up -d`.

The `ARG WYNN_TOOLS_REF` default in [docker/Dockerfile](docker/Dockerfile)
is only used as the last-resort fallback when the GitHub API call fails
AND for plain `docker build` invocations without `--build-arg`. CI
overrides it on every run, so don't expect bumping it to change what
production runs.

### Regenerating a patch against a broken upstream

```bash
git clone https://github.com/wynn-tools/wynn.tools.git /tmp/wt
cd /tmp/wt
git checkout <upstream-sha-from-failed-run>
git apply --reject /path/to/item-renderer/patches/000X-foo.patch
# resolve *.rej hunks by hand, delete the *.rej files, then:
git add -A          # -A, not -u: most patches create new files
git diff --cached > /path/to/item-renderer/patches/000X-foo.patch
```
Open a PR against item-renderer with the regenerated patch. The
`upstream-track` workflow's PR run will re-verify against upstream
main; on merge the main-push run publishes fresh `:<sha>` + `:rolling`
tags.

## Don't

- Don't add upstream wynn.tools source into this repo. The whole
  structural point is that they live only in upstream.
- Don't expose the sidecar publicly via Traefik — it's `verify`-only and
  has no auth.
- Don't bypass the patches by editing the running container. Patches are
  the source of truth.

## License

AGPLv3, inherited from upstream wynn.tools — see [LICENSE](LICENSE).
