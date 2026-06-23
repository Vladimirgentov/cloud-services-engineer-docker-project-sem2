# syntax=docker/dockerfile:1.7

ARG GO_VERSION=1.22
ARG ALPINE_VERSION=3.20

FROM golang:${GO_VERSION}-alpine AS builder

WORKDIR /src

RUN apk add --no-cache git ca-certificates

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . .

RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /out/backend ./cmd/api

FROM alpine:${ALPINE_VERSION}

RUN apk add --no-cache ca-certificates wget \
    && addgroup -S app \
    && adduser -S -G app -u 10001 app

WORKDIR /app

COPY --from=builder /out/backend /app/backend

RUN chown -R app:app /app

USER 10001:10001

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8081/health || exit 1

ENTRYPOINT ["/app/backend"]