# run-migrations.sh

> Runs database migrations inside a Kubernetes Job. Used in CI/CD pipelines after build, before deploy.

## What It Does

1. Creates a short-lived Kubernetes Job in the target namespace
2. Mounts database credentials from External Secrets
3. Runs the migration command (Flyway, Alembic, Prisma, etc.)
4. Waits for completion with timeout
5. Captures logs and cleans up the Job

## Usage

```bash
./ci/run-migrations.sh \
  --service payments-api \
  --namespace payments \
  --cluster eks-staging-01 \
  --image 123456789.dkr.ecr.us-east-1.amazonaws.com/payments-api:a1b2c3d
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--service` | Yes | Service name |
| `--namespace` | Yes | Kubernetes namespace |
| `--cluster` | Yes | Cluster context |
| `--image` | Yes | Docker image with migration files |
| `--command` | No | Migration command (default: `npm run migrate`) |
| `--timeout` | No | Timeout in seconds (default: 300) |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 --service <name> --namespace <ns> --cluster <ctx> --image <img> [--command <cmd>] [--timeout <s>]"
  exit 1
}

SERVICE="" NAMESPACE="" CLUSTER="" IMAGE="" COMMAND="npm run migrate" TIMEOUT=300
while [[ $# -gt 0 ]]; do
  case $1 in
    --service) SERVICE="$2"; shift 2 ;;
    --namespace) NAMESPACE="$2"; shift 2 ;;
    --cluster) CLUSTER="$2"; shift 2 ;;
    --image) IMAGE="$2"; shift 2 ;;
    --command) COMMAND="$2"; shift 2 ;;
    --timeout) TIMEOUT="$2"; shift 2 ;;
    *) usage ;;
  esac
done
[[ -z "$SERVICE" || -z "$NAMESPACE" || -z "$CLUSTER" || -z "$IMAGE" ]] && usage

JOB_NAME="${SERVICE}-migrate-$(date +%s)"

echo "==> Running migration: $JOB_NAME"

cat <<EOF | kubectl --context "$CLUSTER" apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: ${JOB_NAME}
  namespace: ${NAMESPACE}
  labels:
    app: ${SERVICE}
    type: migration
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: ${IMAGE}
          command: ["sh", "-c", "${COMMAND}"]
          envFrom:
            - secretRef:
                name: ${SERVICE}-db-creds
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
EOF

echo "==> Waiting for completion (timeout: ${TIMEOUT}s)"
kubectl --context "$CLUSTER" -n "$NAMESPACE" wait \
  --for=condition=complete \
  --timeout="${TIMEOUT}s" \
  "job/${JOB_NAME}" 2>/dev/null && STATUS="success" || STATUS="failed"

echo "==> Migration logs:"
kubectl --context "$CLUSTER" -n "$NAMESPACE" logs "job/${JOB_NAME}" || true

if [[ "$STATUS" == "success" ]]; then
  echo "[✓] Migration completed successfully"
else
  echo "[✗] Migration failed — check logs above"
  exit 1
fi
```

## Related

- [Docker build](./docker-build.md) — runs before migrations in the pipeline
- [Secrets management](https://github.com/naamabenoliel/platform-techdocs/blob/main/security/secrets-management.md) — how db creds are mounted
