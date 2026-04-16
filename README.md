# aws-terraform-action

> **Complete Terraform CI/CD GitHub Action for AWS**
>
> One action handles the full lifecycle: format check → validate → lint →
> docs → security scan → plan → apply.

[![Test Action](https://github.com/CloudNinjaDev/aws-terraform-action/actions/workflows/test.yml/badge.svg)](https://github.com/CloudNinjaDev/aws-terraform-action/actions/workflows/test.yml)

<div align="center">
  <a href="https://github.com/sponsors/CloudNinjaDev">
    <img src="https://img.shields.io/badge/Sponsor-♥-ff69b4?style=for-the-badge&logo=github-sponsors" alt="Sponsor CloudNinjaDev">
  </a>
</div>

---

## Features

| Stage | Tool | Toggle input |
|-------|------|--------------|
| Format check | `terraform fmt` | `run_fmt` |
| Validate | `terraform validate` | `run_validate` |
| Lint | [TFLint](https://github.com/terraform-linters/tflint) | `run_lint` |
| Documentation | [terraform-docs](https://terraform-docs.io/) | `run_docs` |
| Security scan | [Checkov](https://www.checkov.io/) | `run_security_scan` |
| Plan | `terraform plan` | `run_plan` |
| Apply | `terraform apply` | `run_apply` |
| PR comment | `actions/github-script` | `comment_on_pr` |

---

## Usage

### Minimal CI workflow (Pull Requests)

```yaml
name: Terraform CI

on:
  pull_request:
    branches: [ main ]

permissions:
  contents: read
  pull-requests: write   # Required for PR comments

jobs:
  terraform-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: CloudNinjaDev/aws-terraform-action@v1
        with:
          working_directory: infra/
          run_fmt: 'true'
          run_validate: 'true'
          run_lint: 'true'
          run_security_scan: 'true'
          run_plan: 'true'
          comment_on_pr: 'true'
          github_token: ${{ secrets.GITHUB_TOKEN }}
          # AWS credentials (static keys)
          aws_region: us-east-1
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### CD workflow (Deploy on merge)

```yaml
name: Terraform CD

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  id-token: write   # Required for OIDC

jobs:
  terraform-deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: CloudNinjaDev/aws-terraform-action@v1
        with:
          working_directory: infra/
          run_fmt: 'false'
          run_validate: 'true'
          run_lint: 'false'
          run_security_scan: 'false'
          run_plan: 'true'
          run_apply: 'true'
          auto_approve: 'true'
          comment_on_pr: 'false'
          # AWS credentials via OIDC (recommended)
          aws_region: us-east-1
          role_to_assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          # Terraform options
          var_file: envs/prod.tfvars
          backend_config: 'bucket=my-tfstate,key=prod/terraform.tfstate,region=us-east-1'
```

### Full CI + CD pipeline

```yaml
name: Terraform CI/CD

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  ci:
    name: CI
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Terraform CI
        id: ci
        uses: CloudNinjaDev/aws-terraform-action@v1
        with:
          working_directory: infra/
          terraform_version: '1.7.5'
          run_fmt: 'true'
          run_validate: 'true'
          run_lint: 'true'
          run_docs: 'true'
          run_security_scan: 'true'
          run_plan: 'true'
          comment_on_pr: ${{ github.event_name == 'pull_request' && 'true' || 'false' }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          aws_region: us-east-1
          role_to_assume: arn:aws:iam::123456789012:role/GitHubActionsRole

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: ci
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Terraform Apply
        uses: CloudNinjaDev/aws-terraform-action@v1
        with:
          working_directory: infra/
          terraform_version: '1.7.5'
          run_fmt: 'false'
          run_validate: 'false'
          run_lint: 'false'
          run_plan: 'true'
          run_apply: 'true'
          auto_approve: 'true'
          comment_on_pr: 'false'
          aws_region: us-east-1
          role_to_assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          var_file: envs/prod.tfvars
```

---

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `terraform_version` | Terraform version to install | `latest` |
| `tflint_version` | TFLint version to install | `latest` |
| `terraform_docs_version` | terraform-docs version | `latest` |
| `working_directory` | Terraform root module directory | `.` |
| `run_fmt` | Run `terraform fmt -check` | `true` |
| `run_validate` | Run `terraform validate` | `true` |
| `run_lint` | Run TFLint | `true` |
| `run_docs` | Run terraform-docs | `false` |
| `docs_push` | Push generated docs back to branch | `false` |
| `run_security_scan` | Run Checkov security scan | `true` |
| `run_plan` | Run `terraform plan` | `false` |
| `run_apply` | Run `terraform apply` | `false` |
| `auto_approve` | Pass `-auto-approve` to apply | `false` |
| `aws_region` | AWS region | `us-east-1` |
| `aws_access_key_id` | AWS Access Key ID | `""` |
| `aws_secret_access_key` | AWS Secret Access Key | `""` |
| `aws_session_token` | AWS Session Token | `""` |
| `role_to_assume` | IAM role ARN for OIDC | `""` |
| `backend_config` | Comma-separated backend config values | `""` |
| `var_file` | Path to tfvars file | `""` |
| `extra_args` | Extra arguments for plan/apply | `""` |
| `cache_plugins` | Cache Terraform providers and TFLint plugins to speed up subsequent runs | `true` |
| `comment_on_pr` | Post summary comment on PR | `true` |
| `github_token` | GitHub token for PR comments | `""` (pass `secrets.GITHUB_TOKEN`) |

## Outputs

| Output | Description |
|--------|-------------|
| `fmt_outcome` | `terraform fmt` step result |
| `validate_outcome` | `terraform validate` step result |
| `lint_outcome` | TFLint step result |
| `plan_outcome` | `terraform plan` step result |
| `apply_outcome` | `terraform apply` step result |
| `plan_output` | Full text output of `terraform plan` |

---

## Caching

By default (`cache_plugins: 'true'`), the action caches:

| Cache | Path | Cache key |
|-------|------|-----------|
| Terraform providers | `~/.terraform.d/plugin-cache` | OS + Terraform version + `.terraform.lock.hcl` hash |
| TFLint plugins | `~/.tflint.d/plugins` | OS + TFLint version + `.tflint.hcl` hash |

Caching is transparent — providers and plugins are restored on cache hit or downloaded and saved on cache miss. Disable it by setting `cache_plugins: 'false'`.

```yaml
- uses: CloudNinjaDev/aws-terraform-action@v1
  with:
    cache_plugins: 'false'   # disable caching
```

---

## AWS Authentication

### Static credentials (simple, less secure)

```yaml
aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### OIDC / IAM Role (recommended)

Add `id-token: write` to your workflow permissions and provide:

```yaml
role_to_assume: arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME
```

See [Configuring OpenID Connect in AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services).

---

## Examples

See the [`examples/`](examples/) directory for ready-to-use Terraform modules:

- [`examples/basic`](examples/basic) — S3 bucket with versioning, encryption, and public access block.

---

## License

[MIT](LICENSE)
