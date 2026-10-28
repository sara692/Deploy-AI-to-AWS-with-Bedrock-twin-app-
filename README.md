# AI Digital Twin on AWS (Bedrock + Terraform + GitHub Actions)

An AI "digital twin" chat app deployed on AWS. The backend calls Amazon Bedrock for responses and stores conversation memory in S3. The infrastructure is defined with Terraform, and deployments run automatically from GitHub Actions.

## Architecture

```
GitHub Repository
    | push to main / manual run
    v
GitHub Actions (OIDC login, no stored AWS keys)
    | runs scripts/deploy.sh
    v
Terraform (state in S3, lock in DynamoDB)
    |
    +-- CloudFront --> S3 (Next.js static frontend)
    +-- API Gateway (HTTP API) --> Lambda (Python 3.12)
                                      |-- Amazon Bedrock (LLM)
                                      +-- S3 (conversation memory)
```

Each environment (`dev`, `test`, `prod`) is a separate Terraform workspace with its own isolated resources, named `twin-<env>-*`.

## Project structure

```
.
├── backend/              Python API (FastAPI / Lambda handler) and deploy.py (builds the Lambda zip)
├── frontend/             Next.js app (chat UI)
├── memory/               Local conversation memory (git-ignored)
├── terraform/            Infrastructure as Code
│   ├── versions.tf       Providers (default region + us_east_1 alias for ACM)
│   ├── variables.tf      Input variables
│   ├── main.tf           S3, Lambda, API Gateway, CloudFront, IAM, optional Route 53/ACM
│   ├── outputs.tf        URLs and resource names
│   ├── backend.tf        S3 remote state configuration
│   └── terraform.tfvars  Default variable values (dev)
├── scripts/
│   ├── deploy.sh / deploy.ps1     Build and deploy an environment
│   └── destroy.sh / destroy.ps1   Empty buckets and destroy an environment
└── .github/workflows/
    ├── deploy.yml        Deploy on push to main, or manually per environment
    └── destroy.yml       Manual, confirmation-protected teardown
```

## Prerequisites

- AWS account with an IAM user (not root) that has permissions for Lambda, S3, API Gateway, CloudFront, IAM, DynamoDB and Bedrock
- AWS CLI, configured (`aws configure`)
- Terraform >= 1.0
- Node.js 20+ and npm
- Python 3.12 and [uv](https://docs.astral.sh/uv/)
- A GitHub account and repository
- Bedrock access to at least one model in your chosen region

## Configuration

| Setting       | Where                                              | Example                              |
| ------------- | -------------------------------------------------- | ------------------------------------ |
| AWS region    | AWS CLI profile / `DEFAULT_AWS_REGION`             | `eu-north-1`                         |
| Bedrock model | `terraform/terraform.tfvars` -> `bedrock_model_id` | a model ID that works in your region |
| Project name  | `terraform/terraform.tfvars` -> `project_name`     | `twin`                               |

Notes:

- Terraform uses the AWS credentials and region of your CLI profile. Check with `aws sts get-caller-identity` and `aws configure get region`.
- The `us_east_1` provider alias in `versions.tf` must stay in `us-east-1`. CloudFront only accepts ACM certificates from that region.
- Some Bedrock models are only available through a regional inference profile (for example IDs starting with `eu.` or `us.`). Test a model before using it:

```bash
aws bedrock-runtime converse --region <region> --model-id <model-id> \
  --messages '[{"role":"user","content":[{"text":"Hello"}]}]'
```

## Local development

```bash
# Backend
cd backend
uv run uvicorn server:app --reload --port 8000

# Frontend (in another terminal)
cd frontend
npm install
npm run dev
```

The frontend uses `NEXT_PUBLIC_API_URL` and falls back to `http://localhost:8000`.

## Deploy manually

From the project root:

```bash
./scripts/deploy.sh dev      # or: test, prod
```

Windows (PowerShell):

```powershell
.\scripts\deploy.ps1 -Environment dev
```

The script builds the Lambda package, selects or creates the Terraform workspace, applies the infrastructure, builds the frontend with the API URL, and syncs it to S3. URLs are printed at the end.

Useful Terraform commands:

```bash
cd terraform
terraform workspace list
terraform workspace select dev
terraform plan -var="project_name=twin" -var="environment=dev"
```

## Destroy an environment

```bash
./scripts/destroy.sh dev
```

This empties the S3 buckets first (required before deletion) and then runs `terraform destroy`. CloudFront deletion can take 5 to 15 minutes.

## CI/CD with GitHub Actions

### One-time AWS setup

1. **Remote state**: create an S3 bucket `twin-terraform-state-<account-id>` (versioned, encrypted, private) and a DynamoDB table `twin-terraform-locks` (key `LockID`). Create them in the same region you deploy to.
2. **OIDC provider**: `token.actions.githubusercontent.com` with client ID `sts.amazonaws.com`.
3. **IAM role** `github-actions-twin-deploy` that GitHub Actions can assume, with the permissions Terraform needs.

### Important: OIDC subject format

GitHub may send the subject claim in the ID-based format, which includes numeric user and repo IDs:

```
repo:<owner>@<owner-id>/<repo>@<repo-id>:environment:dev
```

If the role's trust policy only contains `repo:<owner>/<repo>:*`, AWS rejects the login with:

```
Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

Make the trust policy match the format GitHub actually sends:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:<owner>@<owner-id>/<repo>@<repo-id>:*"
}
```

To see the exact subject GitHub sent, check CloudTrail:

```bash
aws cloudtrail lookup-events --region <region> \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --max-results 3 --query "Events[].CloudTrailEvent" --output text
```

Look at `userIdentity.userName`. Events can take several minutes to appear.

### Repository secrets

In GitHub: Settings -> Secrets and variables -> Actions.

| Secret               | Value                                                       |
| -------------------- | ----------------------------------------------------------- |
| `AWS_ROLE_ARN`       | `arn:aws:iam::<account-id>:role/github-actions-twin-deploy` |
| `DEFAULT_AWS_REGION` | your region, e.g. `eu-north-1`                              |
| `AWS_ACCOUNT_ID`     | your 12-digit account ID                                    |

The workflow needs these permissions:

```yaml
permissions:
  id-token: write
  contents: read
```

### Workflows

- **Deploy Digital Twin**: runs on every push to `main` (deploys `dev`) and can be run manually for `dev`, `test` or `prod` from the Actions tab.
- **Destroy Environment**: manual only. You must type the environment name to confirm.

## Troubleshooting

| Problem                                                  | Cause and fix                                                                                                                                                                               |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Provider configuration not present` for `aws.us_east_1` | The alias provider was removed or renamed in `versions.tf`. Restore `alias = "us_east_1"` with `region = "us-east-1"`.                                                                      |
| Resources created in the wrong region                    | The default provider has no region and fell back to the CLI profile. Set `region` in the provider block or run `aws configure set region <region>`. Destroy old resources before switching. |
| Bedrock access denied or throttled                       | Model not available in the region, needs an inference profile ID, not enabled, or the account quota is 0. Test with `converse` and check Service Quotas.                                    |
| OIDC "Not authorized"                                    | Trust policy `sub` does not match what GitHub sends. See the OIDC section above.                                                                                                            |
| Terraform state lock                                     | Another run is in progress. If stuck, `terraform force-unlock <LOCK_ID>`.                                                                                                                   |
| Frontend shows old content                               | Invalidate CloudFront: `aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"`.                                                                                            |
| `S3 bucket already exists`                               | Bucket names are global. Change `project_name`.                                                                                                                                             |

## Cost notes

- Destroy environments you are not using.
- Use a small model for dev and test, and keep API throttling low.
- Third-party Bedrock models are billed through AWS Marketplace and may not be covered by credits. Set a budget alert in Billing -> Budgets.
- The state bucket and lock table cost only a few cents per month.

## Security

- Do not commit `.tfstate` files, `.terraform/`, `.env` files or AWS credentials (see `.gitignore`).
- Use an IAM user or role, not root keys.
- GitHub Actions authenticates with short-lived OIDC credentials, so no long-lived AWS keys are stored in GitHub.
- The memory bucket may contain conversation history. Keep it private.

## Auther

Deploy AI to AWS, GCP, Azure, Vercel with MLOps, Bedrock, SageMaker, RAG, Agents, MCP: scalable, secure and observable cource at udemy : https://www.udemy.com/course/generative-and-agentic-ai-in-production
