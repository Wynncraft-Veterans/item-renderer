# item-renderer — patch overlay over Wynn-Tools/wynn.tools.
#
# This Dockerfile clones a configurable ref of upstream wynn.tools, applies
# the patch series in patches/, and builds the resulting Nuxt server (Nitro
# + Takumi WASM OG renderer) into a small Node runtime image. The repo
# holds no upstream code; the build is reproducibly defined by
# (WYNN_TOOLS_REF, patches/).
#
# Build (local default = the last-known-good fallback below):
#   docker build -t item-renderer:local -f docker/Dockerfile .
#
# Build against a specific upstream ref:
#   docker build --build-arg WYNN_TOOLS_REF=<sha-or-branch> \
#                -t item-renderer:test -f docker/Dockerfile .
#
# In CI, .github/workflows/upstream-track.yml overrides WYNN_TOOLS_REF
# with the latest wynn.tools main-branch SHA (queried from the GitHub
# API on every schedule/push). The ARG below is only used as the
# fallback when that API call fails AND when you build locally without
# --build-arg. Keep it pointing at a ref that's known to apply cleanly.
# Upstream wynn.tools publishes no tagged releases — we track commit SHAs.

# syntax=docker/dockerfile:1.7-labs

ARG WYNN_TOOLS_REF=fe8e535eebd7da8d284b4fc0c7ce32b6b0f56fd6

# Stage 1: fetch upstream and apply patches.
# Done in a tiny image so patch-application failure is fast and obvious.
FROM alpine/git:latest AS source
ARG WYNN_TOOLS_REF
WORKDIR /src
RUN git clone https://github.com/wynn-tools/wynn.tools.git . \
 && git checkout "${WYNN_TOOLS_REF}"
COPY patches/ /patches/
RUN set -e; \
    if ls /patches/*.patch >/dev/null 2>&1; then \
        echo "Applying patches against ${WYNN_TOOLS_REF}:"; \
        # Sequential check-then-apply so a later patch can modify files created
        # by an earlier one.  `git apply --check /patches/*.patch` (all at once)
        # verifies against the pre-patch working tree without staging, so a
        # patch that touches an earlier-patch-created file fails the check even
        # though the actual apply would succeed.
        for p in /patches/*.patch; do \
            echo "  $p"; \
            git apply --check "$p"; \
            git apply "$p"; \
        done; \
        # Every file the series adds or changes, for the builder's type check.
        grep -h '^+++ b/' /patches/*.patch | sed 's|^+++ b/||' | sort -u > .patched-files; \
    else \
        echo "No patches found in /patches/ — building unpatched upstream."; \
        : > .patched-files; \
    fi

# Stage 2: build the patched Nuxt server.
# pnpm is pinned: upstream sets no packageManager field, and `pnpm@latest`
# would move to each new major (12.x by 2026-10) without anyone choosing
# to. 11.5.3 is the version this patch series was last verified with.
#
# Before building, type-check every TypeScript file the patches add or
# change against this upstream ref. `nuxt build` doesn't type-check, so
# an upstream change our code depends on (Arethusa 3.0.0 renamed the
# decoder's `roll` to `value`) would otherwise build green and ship cards
# with no stats. Upstream itself isn't type-clean, so only errors in our
# files fail the build, and the --listFiles check makes sure the type
# check actually reached them. TypeScript is installed outside the
# project: inside it, `npx -p typescript tsc` finds upstream's transitive
# copy under node_modules/.pnpm, counts it as installed, and then has no
# `tsc` to run, because pnpm only links direct dependencies' binaries.
#
# The `pnpm install` flags are minimal because the two policy knobs
# (`trustPolicy: permissive` and `dangerouslyAllowAllBuilds: true`)
# live in upstream's `pnpm-workspace.yaml` via patch 0002. Runtime
# CLI overrides were tried first; they work for the outer install but
# don't propagate to the recursive `pnpm install` that pnpm 11 spawns
# from `runDepsStatusCheck` before any script or `pnpm exec`. Putting
# the knobs in the workspace yaml is the only path the recursive
# install honors.
#
# --no-frozen-lockfile is kept: reproducibility is anchored by
# WYNN_TOOLS_REF pinning the upstream tree, not by the lockfile.
FROM node:22-alpine AS builder
WORKDIR /usr/src/app
COPY --from=source /src .
# -f: patched paths such as `server/api/_og/pua/[encoded].get.ts` must not
# glob in the loop below.
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    set -e -f; \
    corepack enable; \
    corepack prepare pnpm@11.5.3 --activate; \
    pnpm install --no-frozen-lockfile; \
    pnpm exec nuxt prepare; \
    npm install --prefix /tmp/tsc --no-audit --no-fund --loglevel=error typescript@5.9.3; \
    /tmp/tsc/node_modules/.bin/tsc --noEmit --listFiles -p .nuxt/tsconfig.server.json > /tmp/tsc.log 2>&1 || true; \
    for f in $(grep '\.ts$' .patched-files); do \
        grep -qF "$f" /tmp/tsc.log || { echo "Type check never reached $f:"; tail -20 /tmp/tsc.log; exit 1; }; \
    done; \
    if grep -F -f .patched-files /tmp/tsc.log | grep 'error TS'; then \
        echo "Type errors in patched files (above): upstream changed something a patch relies on."; \
        exit 1; \
    fi; \
    pnpm build

# Stage 3: minimal runtime. Nitro output is fully self-contained — we only
# need node and the .output/ tree.
FROM node:22-alpine
WORKDIR /usr/src/app
COPY --from=builder /usr/src/app/.output ./.output
ENV NODE_ENV=production \
    NITRO_PORT=3000 \
    NITRO_HOST=0.0.0.0
EXPOSE 3000
USER node
CMD ["node", ".output/server/index.mjs"]
