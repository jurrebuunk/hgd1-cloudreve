# Cloudreve with the patches in patches/ (OpenID Connect login): upstream's
# source at pinned commits, patched, built, and only the binary swapped into
# upstream's own image. Build from the repo root:
#   docker build -f image/Dockerfile image
# CI (.github/workflows/image.yml) publishes it as
# ghcr.io/jurrebuunk/hgd1-cloudreve:$(cat image/VERSION).

# Upstream release and the commits it is built from (cloudreve tag and its
# assets submodule). Bump all three together, see README.md.
ARG CLOUDREVE_VERSION=4.19.1
ARG CLOUDREVE_COMMIT=a8becb9f5b226024c83230bc52857c04966e2c30
ARG FRONTEND_COMMIT=19da0fe1ecd40971fafa813983d769fdce41573c

FROM alpine/git:v2.52.0@sha256:4a0e72d49596a1f5d3701aeedafdadc5c0da4062be4657c7bdc4017387f591cc AS source
ARG CLOUDREVE_COMMIT
ARG FRONTEND_COMMIT
WORKDIR /src
COPY patches /patches
# Fetch by commit, check it is that commit, then apply the patches; a patch
# that no longer applies fails the build.
RUN set -eu; \
    fetch() { \
      git init -q "$1" && cd "$1" && \
      git fetch -q --depth 1 "$2" "$3" && git checkout -q FETCH_HEAD && \
      test "$(git rev-parse HEAD)" = "$3" && cd ..; \
    }; \
    fetch cloudreve https://github.com/cloudreve/cloudreve.git "$CLOUDREVE_COMMIT"; \
    fetch frontend https://github.com/cloudreve/frontend.git "$FRONTEND_COMMIT"; \
    for p in /patches/backend/*.patch; do git -C cloudreve apply --verbose "$p"; done; \
    for p in /patches/frontend/*.patch; do git -C frontend apply --verbose "$p"; done

FROM node:22.23.3-alpine@sha256:0a7108bf6c7bf5de370ffb1a3ed6be93d405b43ff159f681a8d18c0e2bc2e402 AS frontend
ARG CLOUDREVE_VERSION
RUN apk add --no-cache zip
WORKDIR /src/assets
COPY --from=source /src/frontend ./
# As upstream's .build/build-assets.sh: the version must match the backend's.
ENV NODE_OPTIONS=--max-old-space-size=8192
RUN yarn install --frozen-lockfile --network-timeout 1000000 && \
    yarn version --new-version "$CLOUDREVE_VERSION" --no-git-tag-version && \
    yarn run build && \
    cd .. && zip -qr /assets.zip assets/build

FROM golang:1.26.5-alpine@sha256:0178a641fbb4858c5f1b48e34bdaabe0350a330a1b1149aabd498d0699ff5fb2 AS backend
ARG CLOUDREVE_VERSION
ARG CLOUDREVE_COMMIT
ENV CGO_ENABLED=0 GOTOOLCHAIN=local
WORKDIR /src
COPY --from=source /src/cloudreve ./
RUN go mod download
COPY --from=frontend /assets.zip application/statics/assets.zip
RUN go vet ./service/oidclogin/ ./routers/controllers/ && \
    go test ./service/oidclogin/ && \
    go build -trimpath -o /cloudreve -ldflags "-s -w \
      -X 'github.com/cloudreve/Cloudreve/v4/application/constants.BackendVersion=${CLOUDREVE_VERSION}' \
      -X 'github.com/cloudreve/Cloudreve/v4/application/constants.LastCommit=$(echo ${CLOUDREVE_COMMIT} | cut -c1-7)+oidc'" .

# Upstream's image (ffmpeg, vips, libreoffice, aria2, entrypoint) with our binary.
FROM cloudreve/cloudreve:4.19.1@sha256:5c6c5baf7fa5bc6e1bb9a7509c09dde00e80b618d180244eb1d5f7e36b358ea2
LABEL org.opencontainers.image.source="https://github.com/jurrebuunk/hgd1-cloudreve" \
      org.opencontainers.image.description="Cloudreve with OpenID Connect login (patches in image/patches)" \
      org.opencontainers.image.licenses="GPL-3.0-only"
COPY --from=backend /cloudreve /cloudreve/cloudreve
