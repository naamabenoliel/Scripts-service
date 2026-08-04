# health-check.sh

> Checks health endpoints across services in a namespace and posts results to Slack.

## Usage

```bash
./ops/health-check.sh --namespace payments --slack-webhook $WEBHOOK_URL
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--namespace` | Yes | Kubernetes namespace to check |
| `--cluster` | No | Cluster context (default: current context) |
| `--slack-webhook` | No | Slack webhook URL for alerts |
| `--timeout` | No | HTTP timeout per service in seconds (default: 5) |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="" CLUSTER="" WEBHOOK="" TIMEOUT=5
while [[ $# -gt 0 ]]; do
  case $1 in
    --namespace) NAMESPACE="$2"; shift 2 ;;
    --cluster) CLUSTER="--context $2"; shift 2 ;;
    --slack-webhook) WEBHOOK="$2"; shift 2 ;;
    --timeout) TIMEOUT="$2"; shift 2 ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done
[[ -z "$NAMESPACE" ]] && { echo "Usage: $0 --namespace <ns>"; exit 1; }

FAILURES=()
SERVICES=$(kubectl $CLUSTER -n "$NAMESPACE" get svc -o jsonpath='{.items[*].metadata.name}')

for svc in $SERVICES; do
  ENDPOINT="http://${svc}.${NAMESPACE}.svc.cluster.local/healthz"
  
  # Port-forward and check (simplified — in practice use ingress or in-cluster curl pod)
  STATUS=$(kubectl $CLUSTER -n "$NAMESPACE" exec deploy/"$svc" -- \
    curl -s -o /dev/null -w "%{http_code}" --max-time "$TIMEOUT" "http://localhost:8080/healthz" 2>/dev/null || echo "000")
  
  if [[ "$STATUS" == "200" ]]; then
    echo "[✓] $svc — healthy"
  else
    echo "[✗] $svc — HTTP $STATUS"
    FAILURES+=("$svc (HTTP $STATUS)")
  fi
done

# Slack notification on failures
if [[ ${#FAILURES[@]} -gt 0 && -n "$WEBHOOK" ]]; then
  FAIL_LIST=$(printf '• %s\\n' "${FAILURES[@]}")
  curl -s -X POST "$WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\":\"⚠️ Health check failures in *${NAMESPACE}*:\\n${FAIL_LIST}\"}"
  echo "[!] Slack alert sent"
fi

echo "--- ${#FAILURES[@]} failures out of $(echo $SERVICES | wc -w) services ---"
[[ ${#FAILURES[@]} -gt 0 ]] && exit 1 || exit 0
```

## Related

- [Collect diagnostics](./collect-diagnostics.md) — run after health check failures
- [Incident response](https://github.com/naamabenoliel/platform-techdocs/blob/main/runbooks/incident-response.md) — escalation process
