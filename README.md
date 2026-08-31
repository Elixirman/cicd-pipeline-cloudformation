# Project 4 — Multi-Environment CI/CD Pipeline for CloudFormation

A GitHub Actions pipeline that automatically validates and deploys a CloudFormation
stack to dev, staging, or prod — authenticating to AWS via OIDC with no long-lived
credentials stored anywhere.

## Architecture

git push to main --> GitHub Actions triggered
|
OIDC auth (no stored AWS keys)
|
Validate template
|
cloudformation deploy (create or update, auto-detected)
|
Sync site files to S3
|
dev / staging / prod (isolated per-environment resources)


## Why OIDC instead of access keys

Traditional CI/CD setups store a permanent AWS access key + secret as a GitHub Secret.
If that secret ever leaks, it's valid indefinitely until manually rotated. OIDC instead
lets GitHub Actions request short-lived, auto-expiring credentials at runtime by proving
its identity via a signed token — no static keys exist at all.

## One-time AWS setup (already done for this repo)

```bash
# 1. Create the OIDC identity provider (account-wide, one-time)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <current-github-thumbprint>

# 2. Create an IAM role GitHub Actions can assume, trusting only this GitHub account
aws iam create-role \
  --role-name github-actions-cfn-deploy \
  --assume-role-policy-document file://trust-policy.json

# 3. Attach scoped deploy permissions (CloudFormation + S3 + CloudFront, not admin)
aws iam put-role-policy \
  --role-name github-actions-cfn-deploy \
  --policy-name CfnDeployPermissions \
  --policy-document file://deploy-permissions.json
```

## GitHub repo setup

Add one repository secret (Settings > Secrets and variables > Actions):

| Secret name | Value |
|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/github-actions-cfn-deploy` |

## Triggering a deploy

**Automatic:** any push to `main` deploys to `dev`.

**Manual, any environment:** go to the Actions tab > "Deploy CloudFormation Stack" >
"Run workflow" > pick `dev`, `staging`, or `prod` from the dropdown.

## What the pipeline does, step by step

1. Checks out the repo code
2. Authenticates to AWS via OIDC (no stored keys)
3. Validates the CloudFormation template
4. Runs `cloudformation deploy` — creates the stack if it doesn't exist, updates it if it does
5. Syncs `/site` files to the deployed S3 bucket
6. Prints the live CloudFront URL to the workflow log

## Stack contents

Reuses the Project 1 static site template (S3 + CloudFront + OAC), with an added
`EnvironmentType` parameter so dev/staging/prod each get fully isolated, non-colliding
resources from the same template.

## What this project demonstrates

- CI/CD pipeline design with GitHub Actions
- OIDC federation as a secretless AWS authentication pattern
- Multi-environment infrastructure from a single parameterized template
- Least-privilege IAM scoping for automated deployment roles
- The difference between manual (`create-stack`/`update-stack`) and pipeline-friendly
  (`cloudformation deploy`) CLI commands
