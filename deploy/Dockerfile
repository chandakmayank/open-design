ARG NODE_IMAGE=node:24-bookworm-slim
ARG RUNTIME_IMAGE=node:24-bookworm-slim

# =============================================================================
# Build Stage
# =============================================================================
FROM ${NODE_IMAGE} AS build

ARG HTTP_PROXY
ARG HTTPS_PROXY
ARG http_proxy
ARG https_proxy
ARG no_proxy
ARG NO_PROXY

ENV HTTP_PROXY=${HTTP_PROXY}
ENV HTTPS_PROXY=${HTTPS_PROXY}
ENV http_proxy=${http_proxy}
ENV https_proxy=${https_proxy}
ENV no_proxy=${no_proxy}
ENV NO_PROXY=${NO_PROXY}
ENV CI=true

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 \
        make \
        g++ \
        git \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY scripts/postinstall.mjs ./scripts/postinstall.mjs
COPY packages ./packages
COPY tools ./tools
COPY apps/daemon/package.json ./apps/daemon/package.json
COPY apps/web/package.json ./apps/web/package.json
COPY e2e/package.json ./e2e/package.json

RUN corepack enable && \
    corepack prepare pnpm@10.33.2 --activate && \
    pnpm install --frozen-lockfile

COPY apps ./apps

RUN pnpm --filter @open-design/daemon build && \
    pnpm --filter @open-design/web build && \
    pnpm --filter @open-design/daemon deploy --legacy --prod /app/deploy/daemon && \
    pnpm store prune && \
    rm -rf \
        /root/.cache \
        /root/.local/share/pnpm/store \
        /app/deploy/daemon/node_modules/.cache \
        /app/deploy/daemon/node_modules/@types \
        /app/deploy/daemon/node_modules/.pnpm/@types+* \
        /app/deploy/daemon/node_modules/.pnpm/better-sqlite3@*/node_modules/better-sqlite3/deps \
        /app/deploy/daemon/node_modules/.pnpm/better-sqlite3@*/node_modules/better-sqlite3/src && \
    find /app/deploy/daemon/node_modules -type d \( \
        -name test -o \
        -name tests -o \
        -name "__tests__" -o \
        -name docs -o \
        -name doc -o \
        -name example -o \
        -name examples -o \
        -name ".github" \
    \) -prune -exec rm -rf '{}' + && \
    find /app/deploy/daemon/node_modules -type f \( \
        -name "*.md" -o \
        -name "*.markdown" -o \
        -name "*.d.ts" -o \
        -name "*.d.cts" -o \
        -name "*.d.mts" -o \
        -name "*.map" -o \
        -name "*.tsbuildinfo" -o \
        -name "binding.gyp" \
    \) -delete

# Runtime assets
COPY skills ./skills
COPY design-systems ./design-systems
COPY craft ./craft
COPY prompt-templates ./prompt-templates
COPY assets ./assets
COPY plugins/_official ./plugins/_official

# =============================================================================
# Runtime Stage
# =============================================================================
FROM ${RUNTIME_IMAGE}

ARG OPENCODE_VERSION=latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        tini \
        bash \
        git \
        poppler-utils \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*

#
# Create application user
#
RUN groupadd --gid 1001 open-design && \
    useradd \
        --uid 1001 \
        --gid open-design \
        --create-home \
        --home-dir /home/open-design \
        --shell /bin/bash \
        open-design

#
# Install OpenCode globally
#
RUN npm install -g opencode-ai@${OPENCODE_VERSION} && \
    npm cache clean --force && \
    opencode --version

WORKDIR /app

#
# Copy application
#
COPY --from=build --chown=open-design:open-design /app/deploy/daemon ./apps/daemon
COPY --from=build --chown=open-design:open-design /app/apps/web/out ./apps/web/out

COPY --from=build --chown=open-design:open-design /app/skills ./skills
COPY --from=build --chown=open-design:open-design /app/design-systems ./design-systems
COPY --from=build --chown=open-design:open-design /app/craft ./craft
COPY --from=build --chown=open-design:open-design /app/prompt-templates ./prompt-templates

COPY --from=build --chown=open-design:open-design /app/assets/frames ./assets/frames
COPY --from=build --chown=open-design:open-design /app/assets/community-pets ./assets/community-pets

COPY --from=build --chown=open-design:open-design /app/plugins/_official ./plugins/_official

#
# Prepare writable directories
#
RUN mkdir -p \
        /app/.od \
        /home/open-design/.od \
        /home/open-design/.config \
        /home/open-design/.config/opencode \
        /home/open-design/.cache \
        /home/open-design/.cache/opencode \
        /home/open-design/.local/share \
        /home/open-design/.npm && \
    chown -R open-design:open-design \
        /app \
        /home/open-design && \
    git config --system --add safe.directory /app

#
# Runtime environment
#
ENV HOME=/home/open-design
ENV PATH=/usr/local/bin:$PATH

ENV XDG_CONFIG_HOME=/home/open-design/.config
ENV XDG_CACHE_HOME=/home/open-design/.cache
ENV XDG_DATA_HOME=/home/open-design/.local/share

ENV OD_HOME=/home/open-design/.od
ENV TMPDIR=/tmp

ENV NODE_ENV=production
ENV NODE_OPTIONS="--max-old-space-size=192"

ENV OD_BIND_HOST=0.0.0.0
ENV OD_PORT=7456

ENV OPENCODE_DISABLE_AUTOUPDATE=true
ENV OPENCODE_DISABLE_PRUNE=true

#
# Validate runtime dependencies
#
RUN node --version && \
    npm --version && \
    git --version && \
    opencode --version

EXPOSE 7456

USER open-design

ENTRYPOINT ["/usr/bin/tini", "--"]

CMD ["node", "apps/daemon/dist/cli.js", "--no-open"]
