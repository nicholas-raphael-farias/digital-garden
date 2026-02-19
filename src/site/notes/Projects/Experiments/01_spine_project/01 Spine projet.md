---
{"dg-publish":true,"permalink":"/projects/experiments/01-spine-project/01-spine-projet/"}
---

This spine project is a first to practice devops steps

### Overview
Simple nest js api 
ci/cd with github actions
infrastructure automation with terraform

## Process

### Release strategy

There are 2 environments, dev prod, on this version im not using docker so the artifact creation process involves s3
- development: on pushes to development branch an artifact is created with the commit hash associated
- production: is attached to a github release version pushed to main, an artifact is created with the tag version associated 
### Terraform 
- environments
- modules
- S3 buckets
- 2 IAM roles

### Configuration steps
- Setup aws credentials
- Create bucket
- set terraform.tfvars
- run terraform apply dev and copy oidc_provider_arn to prod
- set github secrets
	- AWS_ROLE_ARN_DEV
	- AWS_ROLE_ARN_PROD
	- AWS_REGION
	- ARTIFACTS_BUCKET


### Authentication in aws and github

Traditional
store an AWS access key + secret in GitHub secrets. Problem — those are long-lived credentials. If leaked, anyone can use them until you rotate.

OIDC
GitHub Actions mints a short-lived JWT token for each workflow run. AWS verifies the token's signature, checks the claims, and issues temporary credentials (valid ~1 hour). No stored secrets. Nothing to rotate. Nothing to leak.


On aws

An ARN (Amazon Resource Name) is just a globally unique ID for any AWS resource. Format:
*arn:aws:{service}:{region}:{account-id}:{resource-type}/{resource-name}*.

IAM Policy 

Each IAM role has two parts: a *trust policy* (who can assume it) and a *permissions policy* (what they can do once assumed).

 Trust Policy (Who Can Assume the Role)

  This is the assume_role_policy on each IAM role. It answers: "which external entity is allowed to call AssumeRoleWithWebIdentity and receive temporary
  credentials for this role?"

```json
  {
    "Statement": [{
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "arn:aws:iam::321488448826:oidc-provider/token.actions.githubusercontent.com"
      },
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "<claim pattern>"
        }
      }
    }]
  }
```

###### Action: sts:AssumeRoleWithWebIdentity
This is the specific STS API call that OIDC federation uses. There are other assume-role mechanisms (AssumeRole for cross-account, AssumeRoleWithSAML for SAML
federation) — this one is specifically for OIDC JWT tokens.

###### Principal.Federated
The OIDC provider ARN. This tells AWS: "only accept tokens from this specific registered identity provider." If someone created their own OIDC provider and minted a token with the same claims, it would be rejected because the principal wouldn't match.

- Condition 1: aud (audience)

"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"

**The aud claim in GitHub's JWT specifies who the token is intended for**
GitHub sets this to sts.amazonaws.com when the workflow uses aws-actions/configure-aws-credentials. This condition ensures the token was specifically minted for AWS STS, not for some other service that also accepts GitHub OIDC tokens (like HashiCorp Vault, GCP, etc.).

- Condition 2: sub (subject) — this is the key security boundary

  Dev role:
  "token.actions.githubusercontent.com:sub": "repo:nicholas-raphael-farias/experiment_01_spineproject:ref:refs/heads/development"

  Prod role:
  "token.actions.githubusercontent.com:sub": "repo:nicholas-raphael-farias/experiment_01_spineproject:ref:refs/tags/v*"

  **The sub claim is a string GitHub embeds in the JWT that describes exactly which repo, branch/tag, and event triggered the workflow. This is the most important**
  **security control**:






Permissions Policy (What the Role Can Do)

Once GitHub Actions has assumed the role and received temporary credentials, this policy governs every API call those credentials make.

```json
  {
    "Statement": [{
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetObjectTagging",
        "s3:PutObjectTagging"
      ],
      "Resource": [
        "arn:aws:s3:::spineproject-artifacts",
        "arn:aws:s3:::spineproject-artifacts/dev/*"
      ]
    }]
  }
```

  *S3 has a split between bucket-level and object-level operations. AWS requires you to specify the correct resource type for each action, otherwise the policy*
  *silently doesn't apply.*



  .       OIDC Provider
    arn:aws:iam::321488448826:oidc-provider/token.actions.githubusercontent.com
          │
          │  referenced as Federated principal in:
         ▼
  Trust Policy (assume_role_policy)
          │
          │  attached to:
         ▼
  IAM Role (dev)                              IAM Role (prod)
    arn:...role/spineproject-github-actions-dev  arn:...role/spineproject-github-actions-prod
          │                                            │
          │  has inline policy :             │  has inline policy referencing:
          ▼                                          ▼
  S3 Bucket ARN + dev/* prefix              S3 Bucket ARN + prod/* prefix
    arn:aws:s3:::spineproject-artifacts       arn:aws:s3:::spineproject-artifacts



  And in GitHub:

  GitHub Secret AWS_ROLE_ARN_DEV  = arn:...role/spineproject-github-actions-dev
  GitHub Secret AWS_ROLE_ARN_PROD = arn:...role/spineproject-github-actions-prod
          │
          ▼
  aws-actions/configure-aws-credentials
    role-to-assume: ${{ secrets.AWS_ROLE_ARN_DEV }}
          │
          ▼
  STS validates JWT → checks trust policy → issues temp credentials
          │
          ▼
  aws s3 cp → checked against permissions policy → allowed or denied



### Steps
i ran the terraform init and terraform plan and terrafom apply on develop 

out dev
artifacts_bucket_arn = "arn:aws:s3:::spineproject-artifacts"
artifacts_bucket_name = "spineproject-artifacts"
github_actions_role_arn = "arn:aws:iam::321488448826:role/spineproject-github-actions-dev"
oidc_provider_arn = "arn:aws:iam::321488448826:oidc-provider/token.actions.githubusercontent.com"

out prod
github_actions_role_arn = "arn:aws:iam::321488448826:role/spineproject-github-actions-prod"


### Errors


copy failed: s3://spineproject-artifacts/dev/5b24803015206e7ced7eac2e499b7b53b2feea16.zip to s3://spineproject-artifacts/dev/latest.zip An error occurred (AccessDenied) when calling the GetObjectTagging operation: User: arn:aws:sts::321488448826:assumed-role/spineproject-github-actions-dev/GitHubActions is not authorized to perform: s3:GetObjectTagging on resource: "arn:aws:s3:::spineproject-artifacts/dev/5b24803015206e7ced7eac2e499b7b53b2feea16.zip" because no identity-based policy allows the s3:GetObjectTagging action

[](https://github.com/nicholas-raphael-farias/experiment_01_spineproject/actions/runs/21806190962/job/62909869126#step:5:270)Error: Process completed with exit code 1.



In short: aws s3 cp from S3→S3 preserves tags by default, which requires GetObjectTagging on the source and PutObjectTagging on the destination. Our policy only
   grants PutObject, GetObject, and ListBucket.

  Want me to add the two missing actions to the policy? It's a one-line change in terraform/modules/ci-iam/main.tf:

       actions = [
         "s3:PutObject",
         "s3:GetObject",
         "s3:ListBucket",
  +      "s3:GetObjectTagging",
  +      "s3:PutObjectTagging",
       ]



### Terraform 

*backend.tf*
**where terraform stores state**
Every time you run terraform apply, Terraform needs to know what resources it already created (so it can diff against your config and only create/update/destroy what changed). That state is stored as a JSON file called terraform.tfstate

By default it would live on your local disk, which means:
  - If you lose your laptop, you lose track of what Terraform manages
  - Two people can't run Terraform at the same time without conflicts

The S3 backend stores it remotely at **s3://spineproject-terraform-state/dev/terraform.tfstate**. Prod uses the same bucket but a different key (prod/terraform.tfstate), so the two environments have completely independent state — applying dev can never accidentally modify prod resources or vice versa.

This backend bucket (spineproject-terraform-state) must be **created manually** before the first terraform init, because Terraform can't create the bucket it needs
  to store its own state in — chicken-and-egg.


*variables.tf*
**Input declarations**

variable "aws_region" { ... }        # default: "us-east-1"
variable "project_name" { ... }       # no default — must be provided

This is the **interface**. It declares what inputs the environment needs without providing values.
Think of it like a TypeScript interface — it defines the shape, not the data.

These declarations exist so that:
- terraform plan will error if you forget a required value
- Other files in this directory can reference them as var.project_name, etc.
- Terraform knows the type and can validate before applying


*terraform.tfvars*
**The actual values**
This fills in the variables declared in variables.tf. Terraform auto-loads any file named terraform.tfvars in the working directory. **This is the only file that changes between environments** — prod has the same variable names but could have different values.


*main.tf*
**The actual interface**
- Provider configuration
- Module calls
	- Calls the artifacts-bucket module, passing in the bucket name from tfvars. This creates the S3 bucket with versioning, encryption, public access block, and lifecycle rules. The module is reusable — prod could call it too, but we chose to have prod reference the existing bucket instead (single bucket, two prefixes).

data "aws_s3_bucket" "artifacts" {
    bucket = var.artifacts_bucket_name
  }

  This is the key difference from dev. Dev uses module "artifacts_bucket" which creates the bucket. Prod uses data "aws_s3_bucket" which reads an existing bucket.

  **A data source is a read-only lookup** — it queries AWS for the bucket named spineproject-artifacts, fetches its attributes (ARN, region, domain name, etc.), and
  makes them available to other resources. It creates nothing. If the bucket doesn't exist, terraform plan will fail with a "not found" error, which is a safety
  net — you can't accidentally apply prod before dev.

  We need the bucket's ARN to scope the IAM policy, but we don't want prod to own the bucket's lifecycle. If you ever destroy prod's Terraform state, the bucket
  (and all artifacts) survive because prod never created it.


dev creates:                        prod creates:
    ├─ S3 bucket                        ├─ IAM role (prod)
    ├─ OIDC provider                    └─ (that's it)
    ├─ IAM role (dev)
    └─ outputs oidc_provider_arn ──────► consumed as input

  dev owns:                           prod references:
    bucket lifecycle                    bucket via data source
    OIDC provider                       OIDC provider via ARN variable

  Prod is intentionally minimal — it only creates what it uniquely needs (the prod IAM role). Everything shared (bucket, OIDC provider) is owned by dev and
  referenced by prod.

*outputs.tf* 
**Values exported after apply**

After terraform apply finishes, these values are printed to your terminal and stored in state. They serve two purposes:

**Cross-environment wiring** — oidc_provider_arn is output here so you can copy it into prod's terraform.tfvars. Prod needs to reference the same OIDC provider without recreating it.

**GitHub secrets setup** — github_actions_role_arn is the value you put into the AWS_ROLE_ARN_DEV GitHub secret so the deploy workflow knows which role to assume.




terraform init      ← downloads AWS provider, connects to S3 backend
  terraform plan      ← reads .tfvars, diffs desired state vs actual, shows what will change
  terraform apply     ← executes in order:
	  1. S3 bucket (artifacts-bucket module)
	  2. OIDC provider + IAM role (ci-iam module, waits for bucket ARN)
	prints outputs:
	  → bucket name/ARN
	  → role ARN (→ goes into GitHub secrets)
	  → OIDC provider ARN (→ goes into prod tfvars)
	  
## Notes

- Terraform commands

1. .terraform/ directories — These contain downloaded providers (the ~400MB terraform-provider-aws binary), local state, and module cache. They should never be
  committed — they're equivalent to node_modules/.
  2. AWSCLIV2.pkg — This is the AWS CLI installer binary. Definitely shouldn't be in the repo.