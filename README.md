# terraform-static-website-public
Terraform project for hosting a static website on AWS with: S3, CloudFront, ACM, Route 53, WAF, Github Actions

- **S3** — primary bucket (us-east-1) + DR bucket (us-west-2) with cross-region replication
- **CloudFront** — global CDN with origin group failover (primary → DR on 4xx/5xx), OAC, TLS 1.2
- **ACM** — DNS-validated certificate in us-east-1
- **Route53** — alias A/AAAA records pointing to CloudFront (data source for existing hosted zone)
- **WAF** — WebACL with AWSManagedRulesCommonRuleSet, CloudWatch logging
- **GitHub Actions** — Checkov, Infracost, and `terraform plan` on PRs; apply with manual approval gate for prod

## Project Structure

```
terraform-static-website/
├── .github/workflows/
│   ├── terraform-pr.yml      # PR: Checkov + Infracost + plan (dev & prod)
│   └── terraform-apply.yml   # Merge: apply dev → approval gate → apply prod
├── .checkov.yml               # Checkov skip list
├── bootstrap/                 # One-time: OIDC provider, IAM roles, Secrets Manager
├── modules/
│   ├── s3/                    # Single-bucket module (call twice: primary + DR)
│   ├── cloudfront/            # Distribution + OAC + origin group
│   ├── acm/                   # Certificate + DNS validation
│   ├── route53/               # Alias records (data source for existing zone)
│   └── waf/                   # WebACL + CloudWatch log group
└── environments/
    ├── dev/                   # dev.example.com, PriceClass_100
    └── prod/                  # example.com + www, PriceClass_All
```

## Prerequisites

1. AWS account with a Route53 hosted zone for your domain
2. An S3 bucket and DynamoDB table for Terraform remote state
3. GitHub **Environments** configured (no AWS credentials stored in GitHub):
   - `dev` — no protection rules (auto-deploy)
   - `prod-plan` — no protection rules (plan only)
   - `production` — add required reviewers for the approval gate
4. GitHub **Variables** (not secrets — these are non-sensitive ARNs):
   - `AWS_OIDC_ROLE_ARN_DEV` — output of `bootstrap` (`dev_role_arn`)
   - `AWS_OIDC_ROLE_ARN_PROD` — output of `bootstrap` (`prod_role_arn`)
   - `INFRACOST_SECRET_NAME` — output of `bootstrap` (`infracost_secret_name`)

## First-Time Setup

### 0. Run the bootstrap module (once per AWS account)

The `bootstrap/` module creates:
- GitHub Actions **OIDC Identity Provider** in IAM — lets GitHub request short-lived AWS tokens without any stored credentials
- **IAM roles** (`github-actions-dev-deploy`, `github-actions-prod-deploy`) with least-privilege permissions
- **Secrets Manager secret** (`ci/infracost-api-key`) — the Infracost API key lives in AWS, not GitHub

First update `bootstrap/backend.tf` with your state bucket and DynamoDB table names, then:

```bash
cd bootstrap

# Set non-sensitive vars as environment variables
export TF_VAR_github_org="pjones-git"
export TF_VAR_github_repo="terraform-static-website"

# Prompt for the Infracost API key interactively — no echo, never stored in shell history
read -rs TF_VAR_infracost_api_key && export TF_VAR_infracost_api_key

terraform init
terraform apply

# Clear the key from the environment when done
unset TF_VAR_infracost_api_key
```

> **Never** pass the Infracost key via `-var="infracost_api_key=..."` — it would be saved in your shell history and visible in `ps aux`.

Copy the outputs to GitHub **Variables** (repo Settings → Secrets and variables → Actions → Variables):

| GitHub Variable | Terraform output |
|---|---|
| `AWS_OIDC_ROLE_ARN_DEV` | `dev_role_arn` |
| `AWS_OIDC_ROLE_ARN_PROD` | `prod_role_arn` |
| `INFRACOST_SECRET_NAME` | `infracost_secret_name` |

> **Note:** These go in **Variables** (not Secrets) — they're non-sensitive ARN strings.
> The actual Infracost API key is in Secrets Manager and is fetched at runtime via the OIDC role.
> No AWS credentials are stored in GitHub at all.

### 1. Bootstrap remote state

Create the S3 bucket and DynamoDB table before running `terraform init`:

```bash
aws s3 mb s3://YOUR_ORG-terraform-state --region us-east-1
aws s3api put-bucket-versioning \
  --bucket YOUR_ORG-terraform-state \
  --versioning-configuration Status=Enabled
aws dynamodb create-table \
  --table-name YOUR_ORG-terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### 2. Configure backend and tfvars

Edit the following files, replacing placeholder values:

- `environments/dev/backend.tf` — set `bucket` and `dynamodb_table`
- `environments/prod/backend.tf` — set `bucket` and `dynamodb_table`
- `environments/dev/terraform.tfvars` — set `domain_name` (and optionally `project`)
- `environments/prod/terraform.tfvars` — set `domain_name` (and optionally `project`)

### 3. Initialize and apply

```bash
# Dev
cd environments/dev
terraform init
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars

# Prod
cd ../prod
terraform init
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars
```

## Deploying Website Content

After infrastructure is provisioned, upload files to the primary S3 bucket:

```bash
# Get the bucket name from Terraform output
BUCKET=$(terraform -chdir=environments/prod output -raw primary_bucket_name)
DIST_ID=$(terraform -chdir=environments/prod output -raw cloudfront_distribution_id)

# Sync your build directory
aws s3 sync ./dist s3://$BUCKET/ --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```

Files are automatically replicated to the DR bucket via S3 Cross-Region Replication.

## DR Failover

CloudFront uses an **origin group** with automatic failover. If the primary S3 origin returns
HTTP 403, 404, 500, 502, 503, or 504, CloudFront automatically routes the request to the DR
bucket in us-west-2. No manual intervention is required.

## Security

| Control | Implementation |
|---|---|
| No public S3 access | `block_public_access` + OAC-only bucket policy |
| TLS 1.2 minimum | `minimum_protocol_version = TLSv1.2_2021` |
| HTTPS redirect | `viewer_protocol_policy = redirect-to-https` |
| WAF | AWSManagedRulesCommonRuleSet |
| Encryption at rest | S3 SSE-AES256 |
| Security response headers | AWS managed SecurityHeadersPolicy |
| State encryption | S3 backend with `encrypt = true` |

## CI/CD Workflow

```
Pull Request to main
  ├─ Checkov scan (fails PR on HIGH/CRITICAL)
  ├─ Infracost cost estimate (PR comment)
  ├─ terraform plan (dev) → PR comment
  └─ terraform plan (prod) → PR comment

Merge to main
  ├─ terraform apply (dev) [automatic]
  ├─ Manual approval gate [GitHub Environment: production]
  └─ terraform apply (prod) [after approval]
```
