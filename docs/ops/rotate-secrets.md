# rotate-secrets.sh

> Rotates secrets in AWS Secrets Manager and triggers External Secrets Operator to sync the new values into Kubernetes.

## Usage

```bash
./ops/rotate-secrets.sh --team payments-squad --secret-path /platform/payments/db-password
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--team` | Yes | Team owning the secret |
| `--secret-path` | Yes | AWS Secrets Manager path |
| `--generate` | No | Auto-generate a new 32-char password |
| `--value` | No | Set a specific value (mutually exclusive with `--generate`) |
| `--dry-run` | No | Show what would change without applying |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

TEAM="" SECRET_PATH="" GENERATE=false VALUE="" DRY_RUN=false
while [[ $# -gt 0 ]]; do
  case $1 in
    --team) TEAM="$2"; shift 2 ;;
    --secret-path) SECRET_PATH="$2"; shift 2 ;;
    --generate) GENERATE=true; shift ;;
    --value) VALUE="$2"; shift 2 ;;
    --dry-run) DRY_RUN=true; shift ;;
    *) echo "Unknown flag: $1"; exit 1 ;;
  esac
done
[[ -z "$TEAM" || -z "$SECRET_PATH" ]] && { echo "Usage: $0 --team <t> --secret-path <p> [--generate|--value <v>]"; exit 1; }

# Verify ownership
TAGS=$(aws secretsmanager describe-secret --secret-id "$SECRET_PATH" --query 'Tags' --output json 2>/dev/null)
OWNER=$(echo "$TAGS" | grep -oP '"Value":\s*"\K[^"]+' | head -1)
if [[ "$OWNER" != "$TEAM" ]]; then
  echo "[✗] Secret owned by '$OWNER', not '$TEAM'. Aborting."
  exit 1
fi

# Generate or use provided value
if [[ "$GENERATE" == true ]]; then
  NEW_VALUE=$(openssl rand -base64 32 | tr -dc 'a-zA-Z0-9' | head -c 32)
  echo "[i] Generated new 32-character password"
elif [[ -n "$VALUE" ]]; then
  NEW_VALUE="$VALUE"
else
  echo "[✗] Must specify --generate or --value"
  exit 1
fi

if [[ "$DRY_RUN" == true ]]; then
  echo "[dry-run] Would update: $SECRET_PATH"
  echo "[dry-run] Would trigger ESO sync in namespace: $TEAM"
  exit 0
fi

# Update in Secrets Manager
aws secretsmanager update-secret \
  --secret-id "$SECRET_PATH" \
  --secret-string "$NEW_VALUE"
echo "[✓] Secret updated in AWS Secrets Manager"

# Force ESO to sync immediately
kubectl annotate externalsecrets -n "$TEAM" --all \
  force-sync="$(date +%s)" --overwrite 2>/dev/null || true
echo "[✓] External Secrets sync triggered for namespace: $TEAM"

# Verify sync (wait up to 60s)
echo "[i] Waiting for sync..."
for i in $(seq 1 12); do
  STATUS=$(kubectl get externalsecrets -n "$TEAM" -o jsonpath='{.items[0].status.conditions[0].status}' 2>/dev/null)
  if [[ "$STATUS" == "True" ]]; then
    echo "[✓] Secrets synced successfully"
    exit 0
  fi
  sleep 5
done
echo "[!] Sync not confirmed within 60s — check ESO logs"
```

## Related

- [Secrets management](https://github.com/naamabenoliel/platform-techdocs/blob/main/security/secrets-management.md) — architecture and setup
- [Crossplane troubleshooting](https://github.com/naamabenoliel/platform-techdocs/blob/main/runbooks/crossplane-troubleshooting.md) — provider creds are rotated with this script too
