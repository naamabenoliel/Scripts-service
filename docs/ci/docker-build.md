# docker-build.sh

> Standardized Docker build used in CI pipelines. Handles multi-stage builds, layer caching, image tagging, and Trivy vulnerability scanning.

## What It Does

1. Builds the Docker image with BuildKit and cache mount
2. Tags with git SHA, branch name, and `latest` (if main)
3. Runs Trivy vulnerability scan — fails on critical CVEs
4. Pushes to ECR

## Usage

```bash
./ci/docker-build.sh \
  --service payments-api \
  --dockerfile Dockerfile \
  --registry 123456789.dkr.ecr.us-east-1.amazonaws.com
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--service` | Yes | Service name (used as image name) |
| `--dockerfile` | No | Path to Dockerfile (default: `./Dockerfile`) |
| `--registry` | Yes | ECR registry URL |
| `--skip-scan` | No | Skip Trivy scan (not recommended) |
| `--push` | No | Push to registry after build (default: true in CI) |

## Tagging Strategy

| Tag | When | Example |
|-----|------|---------|
| Git SHA (short) | Always | `payments-api:a1b2c3d` |
| Branch | Always | `payments-api:feature-auth` |
| `latest` | main branch only | `payments-api:latest` |
| Semver | When git tag exists | `payments-api:v1.2.3` |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 --service <name> --registry <url> [--dockerfile <path>] [--skip-scan]"
  exit 1
}

SERVICE="" REGISTRY="" DOCKERFILE="./Dockerfile" SKIP_SCAN=false
while [[ $# -gt 0 ]]; do
  case $1 in
    --service) SERVICE="$2"; shift 2 ;;
    --registry) REGISTRY="$2"; shift 2 ;;
    --dockerfile) DOCKERFILE="$2"; shift 2 ;;
    --skip-scan) SKIP_SCAN=true; shift ;;
    *) usage ;;
  esac
done
[[ -z "$SERVICE" || -z "$REGISTRY" ]] && usage

SHA=$(git rev-parse --short HEAD)
BRANCH=$(git rev-parse --abbrev-ref HEAD | sed 's/[^a-zA-Z0-9]/-/g')
IMAGE="${REGISTRY}/${SERVICE}"

echo "==> Building ${IMAGE}:${SHA}"

# Build with BuildKit
DOCKER_BUILDKIT=1 docker build \
  --file "$DOCKERFILE" \
  --tag "${IMAGE}:${SHA}" \
  --tag "${IMAGE}:${BRANCH}" \
  --build-arg BUILD_SHA="$SHA" \
  --cache-from "${IMAGE}:latest" \
  .

# Tag latest on main
if [[ "$BRANCH" == "main" ]]; then
  docker tag "${IMAGE}:${SHA}" "${IMAGE}:latest"
fi

# Semver tag if exists
GIT_TAG=$(git describe --tags --exact-match 2>/dev/null || echo "")
if [[ -n "$GIT_TAG" ]]; then
  docker tag "${IMAGE}:${SHA}" "${IMAGE}:${GIT_TAG}"
fi

# Trivy scan
if [[ "$SKIP_SCAN" == false ]]; then
  echo "==> Scanning for vulnerabilities"
  trivy image --severity CRITICAL --exit-code 1 "${IMAGE}:${SHA}"
  echo "[✓] No critical CVEs found"
fi

# Push
echo "==> Pushing to ECR"
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$REGISTRY"
docker push "${IMAGE}" --all-tags

echo "[✓] Pushed: ${IMAGE}:${SHA}"
```

## Related

- [Run migrations](./run-migrations.md) — typically runs after build in the pipeline
- [Security scorecard](https://github.com/naamabenoliel/platform-techdocs/blob/main/architecture/service-catalog.md) — Trivy results feed into security scorecard
