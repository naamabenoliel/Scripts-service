# Scripts-service

> Operational and utility scripts for the platform engineering team.

## Structure

```
scripts-service/
├── onboarding/       # New engineer setup automation
├── ci/               # CI/CD helper scripts
├── ops/              # Operational & incident response scripts
└── utils/            # General-purpose utilities
```

## Scripts

### Onboarding
| Script | Description |
|--------|-------------|
| [setup-dev-env.sh](./onboarding/setup-dev-env.md) | Bootstraps a new engineer's local dev environment |
| [create-namespace.sh](./onboarding/create-namespace.md) | Provisions a team namespace with RBAC and resource quotas |

### CI/CD
| Script | Description |
|--------|-------------|
| [docker-build.sh](./ci/docker-build.md) | Standardized Docker build with caching, tagging, and vulnerability scan |
| [run-migrations.sh](./ci/run-migrations.md) | Database migration runner for CI pipelines |

### Operations
| Script | Description |
|--------|-------------|
| [health-check.sh](./ops/health-check.md) | Multi-service health check with Slack alerting |
| [rotate-secrets.sh](./ops/rotate-secrets.md) | Rotates AWS Secrets Manager secrets and syncs to k8s |
| [collect-diagnostics.sh](./ops/collect-diagnostics.md) | Gathers pod logs, events, and resource usage for incident triage |

### Utilities
| Script | Description |
|--------|-------------|
| [cost-report.py](./utils/cost-report.md) | Generates weekly AWS cost breakdown by team/service |
| [stale-resources.sh](./utils/stale-resources.md) | Finds orphaned cloud resources not linked to any service |

## Usage

All scripts support `--help`. Most require `kubectl`, `aws`, and `jq` installed.

```bash
./ops/health-check.sh --namespace payments --slack-webhook $WEBHOOK_URL
```

## Contributing

1. Add `set -euo pipefail` to all bash scripts
2. Include a `usage()` function
3. Test in dev cluster before merging
