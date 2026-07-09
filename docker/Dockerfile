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
    else \
        echo "No patches found in /patches/ — building unpatched upstream."; \
    fi

# Stage 2: build the patched Nuxt server.
# corepack pulls the pnpm version pinned by upstream's packageManager
# field (if set); otherwise it falls back to the corepack default.
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
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    corepack enable && \
    corepack prepare pnpm@latest --activate && \
    pnpm install --no-frozen-lockfile && \
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
