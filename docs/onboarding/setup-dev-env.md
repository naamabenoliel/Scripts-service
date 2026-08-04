# setup-dev-env.sh

> Bootstraps a new platform engineer's local development environment.

## What It Does

1. Checks prerequisites (`kubectl`, `aws`, `helm`, `docker`, `argocd`)
2. Configures AWS SSO profile for the platform account
3. Updates kubeconfig for dev and staging clusters
4. Installs pre-commit hooks
5. Clones core platform repos
6. Verifies connectivity to all clusters

## Usage

```bash
./onboarding/setup-dev-env.sh --github-user <username>
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--github-user` | Yes | GitHub username for cloning repos |
| `--skip-clone` | No | Skip repo cloning (if already done) |
| `--region` | No | AWS region (default: `us-east-1`) |

## Prerequisites

- macOS or Linux
- Homebrew (macOS) or apt (Linux)
- GitHub SSH key configured

## Example Output

```
[✓] kubectl v1.30.2
[✓] aws-cli v2.15.0
[✓] helm v3.14.0
[✓] docker v26.1.0
[✓] AWS SSO configured for platform-dev
[✓] kubeconfig updated: eks-dev-01
[✓] kubeconfig updated: eks-staging-01
[✓] Cloned: platform-api, infra-terraform, crossplane-compositions
[✓] Pre-commit hooks installed
[✓] All clusters reachable

Setup complete! See onboarding checklist:
https://github.com/naamabenoliel/platform-techdocs/blob/main/onboarding/new-engineer-checklist.md
```

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

REGION="${REGION:-us-east-1}"
REPOS=("platform-api" "infra-terraform" "crossplane-compositions")
CLUSTERS=("eks-dev-01" "eks-staging-01")

usage() {
  echo "Usage: $0 --github-user <username> [--skip-clone] [--region <region>]"
  exit 1
}

check_tool() {
  if command -v "$1" &>/dev/null; then
    echo "[✓] $1 $(command $1 version 2>/dev/null | head -1)"
  else
    echo "[✗] $1 not found. Install it first."
    exit 1
  fi
}

# Parse args
GITHUB_USER=""
SKIP_CLONE=false
while [[ $# -gt 0 ]]; do
  case $1 in
    --github-user) GITHUB_USER="$2"; shift 2 ;;
    --skip-clone) SKIP_CLONE=true; shift ;;
    --region) REGION="$2"; shift 2 ;;
    *) usage ;;
  esac
done
[[ -z "$GITHUB_USER" ]] && usage

# Check prerequisites
for tool in kubectl aws helm docker argocd; do
  check_tool "$tool"
done

# Configure AWS SSO
aws sso login --profile platform-dev 2>/dev/null || \
  aws configure sso --profile platform-dev

# Update kubeconfig
for cluster in "${CLUSTERS[@]}"; do
  aws eks update-kubeconfig --name "$cluster" --region "$REGION" --alias "$cluster"
  echo "[✓] kubeconfig updated: $cluster"
done

# Clone repos
if [[ "$SKIP_CLONE" == false ]]; then
  mkdir -p ~/src/platform
  for repo in "${REPOS[@]}"; do
    if [[ ! -d ~/src/platform/$repo ]]; then
      git clone "git@github.com:acme-platform/$repo.git" ~/src/platform/"$repo"
    fi
  done
  echo "[✓] Cloned: ${REPOS[*]}"
fi

# Pre-commit hooks
pip install pre-commit --quiet
for repo in "${REPOS[@]}"; do
  (cd ~/src/platform/"$repo" && pre-commit install 2>/dev/null) || true
done
echo "[✓] Pre-commit hooks installed"

# Verify connectivity
for cluster in "${CLUSTERS[@]}"; do
  kubectl --context "$cluster" get nodes &>/dev/null && \
    echo "[✓] $cluster reachable" || \
    echo "[✗] $cluster unreachable"
done

echo -e "\nSetup complete!"
```

## Related

- [New engineer checklist](https://github.com/naamabenoliel/platform-techdocs/blob/main/onboarding/new-engineer-checklist.md)
- [Create namespace](./create-namespace.md) — next step after dev env setup
