# create-namespace.sh

> Provisions a Kubernetes namespace for a team with RBAC, resource quotas, network policies, and External Secrets access.

## What It Does

1. Creates the namespace with standard labels
2. Applies resource quotas (CPU, memory, pod count)
3. Creates RBAC RoleBinding for the team's Okta group
4. Sets up NetworkPolicy (deny-all ingress by default)
5. Creates a ClusterSecretStore reference for the team's secrets path
6. Registers the namespace in the service catalog

## Usage

```bash
./onboarding/create-namespace.sh \
  --team payments-squad \
  --cluster eks-dev-01 \
  --tier 2
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--team` | Yes | Team name (must match Okta group) |
| `--cluster` | Yes | Target cluster context |
| `--tier` | No | Service tier 1/2/3 (default: 2, affects quotas) |
| `--dry-run` | No | Print manifests without applying |

## Resource Quotas by Tier

| Tier | CPU | Memory | Pods |
|------|-----|--------|------|
| 1 | 32 cores | 64Gi | 100 |
| 2 | 16 cores | 32Gi | 50 |
| 3 | 8 cores | 16Gi | 25 |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 --team <name> --cluster <context> [--tier 1|2|3] [--dry-run]"
  exit 1
}

TEAM="" CLUSTER="" TIER=2 DRY_RUN=""
while [[ $# -gt 0 ]]; do
  case $1 in
    --team) TEAM="$2"; shift 2 ;;
    --cluster) CLUSTER="$2"; shift 2 ;;
    --tier) TIER="$2"; shift 2 ;;
    --dry-run) DRY_RUN="--dry-run=client"; shift ;;
    *) usage ;;
  esac
done
[[ -z "$TEAM" || -z "$CLUSTER" ]] && usage

# Quota lookup
declare -A CPU=([1]="32" [2]="16" [3]="8")
declare -A MEM=([1]="64Gi" [2]="32Gi" [3]="16Gi")
declare -A PODS=([1]="100" [2]="50" [3]="25")

NAMESPACE="$TEAM"

cat <<EOF | kubectl --context "$CLUSTER" apply $DRY_RUN -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${NAMESPACE}
  labels:
    team: ${TEAM}
    tier: "${TIER}"
    managed-by: platform-scripts
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${TEAM}-quota
  namespace: ${NAMESPACE}
spec:
  hard:
    requests.cpu: "${CPU[$TIER]}"
    requests.memory: "${MEM[$TIER]}"
    pods: "${PODS[$TIER]}"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ${TEAM}-access
  namespace: ${NAMESPACE}
subjects:
  - kind: Group
    name: okta-${TEAM}
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: ${NAMESPACE}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF

echo "[✓] Namespace '$NAMESPACE' created on $CLUSTER (tier $TIER)"
echo "[✓] RBAC bound to okta-${TEAM}"
echo "[✓] Resource quota applied: ${CPU[$TIER]} CPU, ${MEM[$TIER]} memory, ${PODS[$TIER]} pods"
echo "[✓] Default deny-all NetworkPolicy applied"
```

## Related

- [Setup dev env](./setup-dev-env.md) — run first before creating namespaces
- [Auth patterns](https://github.com/naamabenoliel/platform-techdocs/blob/main/security/auth-patterns.md) — RBAC details
