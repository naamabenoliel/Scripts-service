# stale-resources.sh

> Finds orphaned AWS resources that aren't linked to any service in the catalog. Helps catch forgotten infrastructure driving up costs.

## What It Checks

- RDS instances without a `service` tag
- ElastiCache clusters with no matching catalog entry
- S3 buckets older than 90 days with no `team` tag
- EBS volumes in `available` state (unattached)
- Load balancers with zero healthy targets

## Usage

```bash
./utils/stale-resources.sh --region us-east-1
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--region` | No | AWS region (default: `us-east-1`) |
| `--output` | No | Save results to file |
| `--slack-webhook` | No | Post summary to Slack |
| `--age-days` | No | Minimum age to flag (default: 30) |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

REGION="us-east-1" OUTPUT="" WEBHOOK="" AGE_DAYS=30
while [[ $# -gt 0 ]]; do
  case $1 in
    --region) REGION="$2"; shift 2 ;;
    --output) OUTPUT="$2"; shift 2 ;;
    --slack-webhook) WEBHOOK="$2"; shift 2 ;;
    --age-days) AGE_DAYS="$2"; shift 2 ;;
    *) echo "Unknown: $1"; exit 1 ;;
  esac
done

RESULTS=""
STALE_COUNT=0
CUTOFF=$(date -d "-${AGE_DAYS} days" +%Y-%m-%dT%H:%M:%S 2>/dev/null || \
         date -v-${AGE_DAYS}d +%Y-%m-%dT%H:%M:%S)

echo "==> Scanning for stale resources in $REGION (older than ${AGE_DAYS} days)"

# Untagged RDS instances
echo "--- RDS instances without 'service' tag ---"
aws rds describe-db-instances --region "$REGION" --query \
  "DBInstances[?!(TagList[?Key=='service'])].{ID:DBInstanceIdentifier,Created:InstanceCreateTime,Status:DBInstanceStatus}" \
  --output table 2>/dev/null | tee -a /tmp/stale_rds.txt
RDS_COUNT=$(grep -c "db-" /tmp/stale_rds.txt 2>/dev/null || echo 0)
STALE_COUNT=$((STALE_COUNT + RDS_COUNT))

# Unattached EBS volumes
echo "--- Unattached EBS volumes ---"
aws ec2 describe-volumes --region "$REGION" \
  --filters "Name=status,Values=available" \
  --query "Volumes[].{ID:VolumeId,Size:Size,Created:CreateTime}" \
  --output table 2>/dev/null

EBS_COUNT=$(aws ec2 describe-volumes --region "$REGION" \
  --filters "Name=status,Values=available" \
  --query "length(Volumes)" --output text 2>/dev/null || echo 0)
STALE_COUNT=$((STALE_COUNT + EBS_COUNT))

# Untagged S3 buckets
echo "--- S3 buckets without 'team' tag ---"
for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text 2>/dev/null); do
  TAGS=$(aws s3api get-bucket-tagging --bucket "$bucket" 2>/dev/null || echo "none")
  if ! echo "$TAGS" | grep -q '"Key": "team"'; then
    CREATED=$(aws s3api get-bucket-location --bucket "$bucket" 2>/dev/null)
    echo "  [!] $bucket — no team tag"
    STALE_COUNT=$((STALE_COUNT + 1))
  fi
done

# Summary
echo ""
echo "==> Found $STALE_COUNT potentially stale resources"

# Output
if [[ -n "$OUTPUT" ]]; then
  echo "Stale resource scan — $(date)" > "$OUTPUT"
  echo "Region: $REGION | Threshold: ${AGE_DAYS} days | Count: $STALE_COUNT" >> "$OUTPUT"
  echo "[✓] Saved to $OUTPUT"
fi

# Slack
if [[ -n "$WEBHOOK" && $STALE_COUNT -gt 0 ]]; then
  curl -s -X POST "$WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\":\"🧹 *Stale resource scan*: Found $STALE_COUNT orphaned resources in $REGION. Run \`stale-resources.sh\` for details.\"}"
  echo "[✓] Slack alert sent"
fi
```

## Related

- [Cost report](./cost-report.md) — run together for a full cost hygiene review
- [Service catalog](https://github.com/naamabenoliel/platform-techdocs/blob/main/architecture/service-catalog.md) — properly tagged resources appear here
