# collect-diagnostics.sh

> Gathers logs, events, resource usage, and pod status into a single tarball for incident triage.

## Usage

```bash
./ops/collect-diagnostics.sh --namespace payments --service payments-api
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--namespace` | Yes | Kubernetes namespace |
| `--service` | No | Specific service (default: all pods in namespace) |
| `--cluster` | No | Cluster context (default: current) |
| `--lines` | No | Log tail lines per pod (default: 500) |
| `--output` | No | Output directory (default: `/tmp/diagnostics`) |

## What It Collects

```
diagnostics-payments-20260527-1430/
├── pods.txt              # Pod status, restarts, node placement
├── events.txt            # Namespace events (last 1 hour)
├── resource-usage.txt    # CPU/memory from kubectl top
├── describe-pods.txt     # Full pod descriptions
├── logs/
│   ├── payments-api-abc-123.log
│   ├── payments-api-def-456.log
│   └── payments-api-def-456.previous.log   # Previous container (if crashed)
├── hpa.txt               # HPA status
├── pdb.txt               # PodDisruptionBudget status
└── network-policies.txt  # Active network policies
```

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="" SERVICE="" CLUSTER="" LINES=500 OUTPUT="/tmp/diagnostics"
while [[ $# -gt 0 ]]; do
  case $1 in
    --namespace) NAMESPACE="$2"; shift 2 ;;
    --service) SERVICE="$2"; shift 2 ;;
    --cluster) CLUSTER="--context $2"; shift 2 ;;
    --lines) LINES="$2"; shift 2 ;;
    --output) OUTPUT="$2"; shift 2 ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done
[[ -z "$NAMESPACE" ]] && { echo "Usage: $0 --namespace <ns> [--service <svc>]"; exit 1; }

TIMESTAMP=$(date +%Y%m%d-%H%M)
DIR="${OUTPUT}/diagnostics-${NAMESPACE}-${TIMESTAMP}"
mkdir -p "${DIR}/logs"

SELECTOR=""
[[ -n "$SERVICE" ]] && SELECTOR="-l app=$SERVICE"

echo "==> Collecting diagnostics for $NAMESPACE"

# Pod status
kubectl $CLUSTER -n "$NAMESPACE" get pods $SELECTOR -o wide > "${DIR}/pods.txt"
echo "[✓] Pod status"

# Events
kubectl $CLUSTER -n "$NAMESPACE" get events --sort-by=.lastTimestamp > "${DIR}/events.txt"
echo "[✓] Events"

# Resource usage
kubectl $CLUSTER -n "$NAMESPACE" top pods $SELECTOR > "${DIR}/resource-usage.txt" 2>/dev/null || echo "metrics unavailable" > "${DIR}/resource-usage.txt"
echo "[✓] Resource usage"

# Pod descriptions
kubectl $CLUSTER -n "$NAMESPACE" describe pods $SELECTOR > "${DIR}/describe-pods.txt"
echo "[✓] Pod descriptions"

# Logs
PODS=$(kubectl $CLUSTER -n "$NAMESPACE" get pods $SELECTOR -o jsonpath='{.items[*].metadata.name}')
for pod in $PODS; do
  kubectl $CLUSTER -n "$NAMESPACE" logs "$pod" --tail="$LINES" > "${DIR}/logs/${pod}.log" 2>/dev/null || true
  kubectl $CLUSTER -n "$NAMESPACE" logs "$pod" --previous --tail="$LINES" > "${DIR}/logs/${pod}.previous.log" 2>/dev/null || true
done
# Remove empty previous logs
find "${DIR}/logs" -name "*.previous.log" -empty -delete
echo "[✓] Logs (${LINES} lines each)"

# HPA, PDB, NetworkPolicy
kubectl $CLUSTER -n "$NAMESPACE" get hpa -o wide > "${DIR}/hpa.txt" 2>/dev/null || echo "none" > "${DIR}/hpa.txt"
kubectl $CLUSTER -n "$NAMESPACE" get pdb -o wide > "${DIR}/pdb.txt" 2>/dev/null || echo "none" > "${DIR}/pdb.txt"
kubectl $CLUSTER -n "$NAMESPACE" get networkpolicies > "${DIR}/network-policies.txt" 2>/dev/null || echo "none" > "${DIR}/network-policies.txt"
echo "[✓] HPA, PDB, NetworkPolicies"

# Tar it up
TARBALL="${OUTPUT}/diagnostics-${NAMESPACE}-${TIMESTAMP}.tar.gz"
tar -czf "$TARBALL" -C "$OUTPUT" "diagnostics-${NAMESPACE}-${TIMESTAMP}"
echo "==> Saved to: $TARBALL"
echo "==> Size: $(du -h "$TARBALL" | cut -f1)"
```

## Related

- [Health check](./health-check.md) — run health check first to identify failing services
- [Incident response](https://github.com/naamabenoliel/platform-techdocs/blob/main/runbooks/incident-response.md) — share the tarball in the incident Slack thread
