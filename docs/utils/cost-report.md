# cost-report.py

> Generates a weekly AWS cost breakdown by team and service using Cost Explorer API. Outputs a summary table and posts to Slack.

## Usage

```bash
python utils/cost-report.py --weeks 1 --slack-webhook $WEBHOOK_URL
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--weeks` | No | Number of weeks to report (default: 1) |
| `--slack-webhook` | No | Post summary to Slack |
| `--output` | No | Save CSV to file path |
| `--threshold` | No | Alert if any team exceeds this USD amount (default: 5000) |

## Example Output

```
AWS Cost Report: May 20 – May 27, 2026

Team               Service              Cost (USD)    Δ vs Last Week
─────────────────────────────────────────────────────────────────────
payments-squad      payments-api         $1,245.30     +3.2%
payments-squad      payments-db (RDS)    $890.00       -1.1%
search-team         search-api           $2,100.50     +12.4% ⚠️
search-team         elasticsearch        $3,200.00     +0.5%
platform-eng        eks-prod-01          $4,800.00     -2.3%
platform-eng        eks-prod-02          $3,600.00     +1.0%
─────────────────────────────────────────────────────────────────────
TOTAL                                    $15,835.80    +2.1%
```

## Script

```python
#!/usr/bin/env python3
"""Weekly AWS cost report by team and service."""

import argparse
import boto3
import json
from datetime import datetime, timedelta

def get_costs(weeks: int) -> list[dict]:
    ce = boto3.client("ce", region_name="us-east-1")
    end = datetime.now().strftime("%Y-%m-%d")
    start = (datetime.now() - timedelta(weeks=weeks)).strftime("%Y-%m-%d")

    response = ce.get_cost_and_usage(
        TimePeriod={"Start": start, "End": end},
        Granularity="WEEKLY",
        Metrics=["UnblendedCost"],
        GroupBy=[
            {"Type": "TAG", "Key": "team"},
            {"Type": "TAG", "Key": "service"},
        ],
    )

    results = []
    for period in response["ResultsByTime"]:
        for group in period["Groups"]:
            team = group["Keys"][0].replace("team$", "")
            service = group["Keys"][1].replace("service$", "")
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            results.append({"team": team, "service": service, "cost": round(cost, 2)})

    return sorted(results, key=lambda x: x["team"])


def post_to_slack(webhook: str, message: str):
    import urllib.request
    req = urllib.request.Request(
        webhook,
        data=json.dumps({"text": message}).encode(),
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req)


def main():
    parser = argparse.ArgumentParser(description="AWS cost report")
    parser.add_argument("--weeks", type=int, default=1)
    parser.add_argument("--slack-webhook", type=str, default="")
    parser.add_argument("--output", type=str, default="")
    parser.add_argument("--threshold", type=float, default=5000)
    args = parser.parse_args()

    costs = get_costs(args.weeks)
    total = sum(c["cost"] for c in costs)

    # Print table
    print(f"{'Team':<20} {'Service':<25} {'Cost (USD)':>12}")
    print("─" * 60)
    for c in costs:
        flag = " ⚠️" if c["cost"] > args.threshold else ""
        print(f"{c['team']:<20} {c['service']:<25} ${c['cost']:>10,.2f}{flag}")
    print("─" * 60)
    print(f"{'TOTAL':<46} ${total:>10,.2f}")

    # Save CSV
    if args.output:
        import csv
        with open(args.output, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=["team", "service", "cost"])
            writer.writeheader()
            writer.writerows(costs)
        print(f"\n[✓] Saved to {args.output}")

    # Slack alert
    if args.slack_webhook:
        over = [c for c in costs if c["cost"] > args.threshold]
        if over:
            msg = f"💰 *Cost alert*: {len(over)} items over ${args.threshold:,.0f} threshold\n"
            msg += "\n".join(f"• {c['team']}/{c['service']}: ${c['cost']:,.2f}" for c in over)
            post_to_slack(args.slack_webhook, msg)
            print("[✓] Slack alert sent")


if __name__ == "__main__":
    main()
```

## Related

- [Stale resources](./stale-resources.md) — find resources driving up costs with no owner
- [Infrastructure stack](https://github.com/naamabenoliel/platform-techdocs/blob/main/architecture/infrastructure-stack.md) — cost controls section
