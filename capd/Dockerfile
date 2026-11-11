# syntax=docker/dockerfile:1
ARG BUN_VERSION=1.4.4
ARG DUCKDB_VERSION=v1.1.3

# --- Stage 1: Compile DuckDB from source without AVX2 ---
FROM oven/bun:${BUN_VERSION}-slim AS duckdb-builder
ARG DUCKDB_VERSION

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    git \
    python3 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
RUN git clone --depth 1 --branch ${DUCKDB_VERSION} https://github.com/duckdb/duckdb.git .

# Enforce generic x86-64 and explicitly disable AVX2
ENV CXXFLAGS="-march=x86-64 -mtune=generic -mno-avx2 -O3"
ENV CFLAGS="-march=x86-64 -mtune=generic -mno-avx2 -O3"

# Build DuckDB Node extension from source
WORKDIR /build/tools/nodejs
RUN cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_FLAGS="${CXXFLAGS}" \
    -DCMAKE_C_FLAGS="${CFLAGS}"
RUN cmake --build build --config Release

# --- Stage 2: Build Dashboard ---
FROM oven/bun:${BUN_VERSION}-slim AS dashboard
WORKDIR /usr/src/app
COPY capd/package.json capd/bun.lock ./
RUN bun install --frozen-lockfile
COPY capd/vite.config.mjs ./
COPY capd/dashboard dashboard
RUN bun run build

# --- # Stage 3: Dependencies & Binary Injection ---
FROM oven/bun:${BUN_VERSION}-slim AS deps
WORKDIR /usr/src/app

COPY capd/package.json capd/bun.lock ./
# Let bun install download the standard packages (satisfying all module resolution)
RUN bun install --frozen-lockfile --production

# OVERWRITE the downloaded AVX2 binary with our custom-compiled AVX2-free binary
COPY --from=duckdb-builder /build/tools/nodejs/build/duckdb.node /usr/src/app/node_modules/@duckdb/node-bindings-linux-x64/duckdb.node

# Strip debug symbols to keep image size small
RUN apt-get update && apt-get install -y --no-install-recommends binutils && \
    strip -s /usr/src/app/node_modules/@duckdb/node-bindings-linux-x64/duckdb.node && \
    apt-get purge -y binutils && rm -rf /var/lib/apt/lists/*

COPY core/package.json /tmp/capjs-core/package.json
COPY core/src /tmp/capjs-core/src
RUN rm -rf node_modules/capjs-core \
 && mv /tmp/capjs-core node_modules/capjs-core

# --- Stage 4: Final Release Image ---
FROM oven/bun:${BUN_VERSION}-slim AS release
LABEL org.opencontainers.image.source="https://github.com/tiagozip/cap" \
  org.opencontainers.image.url="https://trycap.dev" \
  org.opencontainers.image.title="cap" \
  org.opencontainers.image.description="capd, the Cap server: a lightweight, open-source proof-of-work captcha" \
  org.opencontainers.image.licenses="Apache-2.0"

ENV NODE_ENV=production \
  SERVER_PORT=3000 \
  DATA_DIR=/usr/src/app/data

COPY --chown=bun:bun capd/package.json ./
COPY --from=deps --chown=bun:bun /usr/src/app/node_modules node_modules
COPY --chown=bun:bun capd/src src
COPY --chown=bun:bun capd/public public
COPY --from=dashboard --chown=bun:bun /usr/src/app/public/dist public/dist
RUN mkdir -p data && chown bun:bun data

USER bun
VOLUME ["/usr/src/app/data"]
EXPOSE 3000/tcp

HEALTHCHECK --interval=15s --timeout=5s --start-period=20s --retries=3 \
  CMD ["bun", "-e", "fetch(`http://127.0.0.1:${process.env.SERVER_PORT || 3000}/health`).then((r) => process.exit(r.ok ? 0 : 1), () => process.exit(1))"]

ENTRYPOINT ["bun", "run", "./src/index.js"]
