# GitHub Actions → AWS OIDC Provider + Role (CloudFormation)

OIDC is a standard for federated identity and can be used with many identity providers and CI platforms. This project specifically configures AWS to trust **GitHub Actions**. It creates or reuses the GitHub OIDC provider and creates an **IAM role** in your AWS account using CloudFormation deployed from a GitHub Actions workflow. Fork or clone it, add the AWS bootstrap repository secrets, click **Run workflow**, and see redacted ARNs in the run summary.

Once set up, your other GitHub workflows can authenticate to AWS **without long-lived access keys**.

---

## What you get

| Resource | Description |
|---|---|
| `AWS::IAM::OIDCProvider` | Creates the GitHub provider, unless an existing provider ARN is supplied |
| `AWS::IAM::Role` | A role that only your chosen GitHub org/repo can assume via OIDC |

**Outputs:** The workflow masks the full provider and role ARNs from logs and shows both in the run summary with the 12-digit AWS account ID redacted. Full ARNs remain available in the CloudFormation stack Outputs.

## OIDC and credential lifetime

The GitHub OIDC token is a short-lived JWT, valid for **5 minutes**. It is exchanged for temporary AWS STS credentials; the OIDC token and AWS credentials have separate expiration times.

The deployment input `role_session_duration_minutes` sets the desired AWS session duration from **15 to 720 minutes** (default: **60**). The workflow sets the IAM role's maximum session duration to at least 60 minutes; AWS requires that role maximum to be between 1 and 12 hours. The GitHub OIDC token itself remains valid for 5 minutes and cannot be extended by this setting.

The role maximum is only a ceiling. In each workflow that assumes the role, set `role-duration-seconds` to the desired minutes multiplied by 60; the credentials action otherwise defaults to 3600 seconds. For example, 90 minutes requires `role-duration-seconds: 5400`.

This template is GitHub-specific because it trusts `https://token.actions.githubusercontent.com` and checks GitHub's `sub` claim. Other OIDC providers require their own provider URL, audience, and trust conditions.

See [GitHub's OIDC token documentation](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token) and [AWS STS AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html).

## Repo structure

```
.
├── cfn/
│   └── github-oidc.yaml              # CloudFormation template
├── .github/
│   └── workflows/
│       └── deploy-oidc.yml           # Deployment workflow (manual trigger)
├── LICENSE                           # MIT License
└── README.md
```

---

## Quick start

### 1. Fork or clone this repo

Click **Fork** on GitHub, or:

```bash
git clone https://github.com/<your-user>/<this-repo>.git
```

> If you clone and push to your own repo, make sure Actions are enabled in it (Settings → Actions → General).

### 2. Create an IAM user for the first deployment

You need one-time credentials to create the IAM role and, unless reusing an existing provider, the OIDC provider (chicken-and-egg: OIDC doesn't exist yet). Create an IAM user with programmatic access and attach a policy like the one below.

<details>
<summary>Suggested deployer policy</summary>

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DeleteStack",
        "cloudformation:DescribeStacks",
        "cloudformation:DescribeStackEvents",
        "cloudformation:CreateChangeSet",
        "cloudformation:DescribeChangeSet",
        "cloudformation:ExecuteChangeSet",
        "cloudformation:DeleteChangeSet",
        "cloudformation:GetTemplateSummary",
        "cloudformation:ValidateTemplate"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateOpenIDConnectProvider",
        "iam:GetOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider",
        "iam:TagOpenIDConnectProvider",
        "iam:AddClientIDToOpenIDConnectProvider",
        "iam:RemoveClientIDFromOpenIDConnectProvider",
        "iam:UpdateOpenIDConnectProviderThumbprint",
        "iam:CreateRole",
        "iam:GetRole",
        "iam:DeleteRole",
        "iam:TagRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:ListAttachedRolePolicies",
        "iam:ListRolePolicies",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
```

Tighten `Resource` to specific ARNs if you like.
</details>

### 3. Add GitHub repository secrets

Go to **Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key ID of the IAM user |
| `AWS_SECRET_ACCESS_KEY` | Secret access key of the IAM user |

> ⚠️ **Never** commit keys to the repo and never pass them as workflow inputs. Workflow inputs are visible in plain text; secrets are masked.

### 4. Run the workflow

**Actions → Deploy GitHub OIDC Provider → Run workflow**, then fill in:

| Input | Description | Default |
|---|---|---|
| `aws_region` | Region to deploy the stack in | `us-east-1` |
| `stack_name` | CloudFormation stack name | `github-oidc` |
| `github_org` | Your GitHub org or username | *(required)* |
| `github_repo` | Repo allowed to assume the role; `*` explicitly allows all repos in the org | *(required)* |
| `existing_oidc_provider_arn` | Existing GitHub OIDC provider ARN; leave blank to create one | *(empty)* |
| `github_subject` | OIDC subject suffix to restrict refs or environments | `ref:refs/heads/main` |
| `role_name` | Name of the IAM role | `github-actions-oidc-role` |
| `role_managed_policy_arns` | Comma-separated managed policy ARNs to attach to the role | `arn:aws:iam::aws:policy/AdministratorAccess` |
| `role_session_duration_minutes` | AWS session duration requested by workflows (15-720 minutes) | `60` |

> ⚠️ `AdministratorAccess` grants full AWS account permissions. If selected, the workflow pauses before accessing AWS and requires approval from a reviewer configured for the `administrator-access` GitHub Environment. Choose narrower policies whenever possible.

To enable the approval gate, open **Settings → Environments → New environment**, create an environment named `administrator-access`, and configure **Required reviewers**. Without required reviewers configured, GitHub will not pause the deployment for human approval. The approval is requested before the job receives AWS credentials.

To combine managed policies, enter comma-separated ARNs, for example:

```text
arn:aws:iam::aws:policy/AmazonEC2FullAccess,arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

`github_repo` is required. Enter a specific repository name; use `*` only if you intend to trust every repository in the organization.

If the AWS account already has a GitHub OIDC provider, set `existing_oidc_provider_arn` to its ARN. The provider must belong to the same AWS account and use `https://token.actions.githubusercontent.com` with `sts.amazonaws.com` as a client ID. When supplied, this stack reuses the provider and does not manage it: its settings and existing roles' trust policies are left unchanged. This stack changes only the role it creates or updates. Only use this input for a provider external to this stack; if this stack created the provider, leave the input blank so CloudFormation continues managing it.

To list provider ARNs, run `aws iam list-open-id-connect-providers --query "OpenIDConnectProviderList[].Arn" --output text` and select the GitHub provider ARN.

You can use AWS-managed policies or existing customer-managed policy ARNs. IAM roles support up to 10 attached managed policies by default.

### 5. Get your ARNs

The deployment workflow masks the full ARNs from logs and displays them in the run summary with the account ID redacted. The redacted values are for identifying the resources, not for use in workflows. Retrieve the full ARNs from **CloudFormation → Stacks → your stack → Outputs** or locally with the AWS CLI:

```bash
aws cloudformation describe-stacks --stack-name github-oidc \
  --region us-east-1 \
  --query "Stacks[0].Outputs" --output table
```

Replace the stack name or region if you used different values. The ARNs identify AWS resources but are not credentials; keep the role trust policy and attached permissions properly restricted.

---

## Using the role in your other workflows

Workflows that use the role do not need IAM user keys. In any workflow in the allowed repo:

```yaml
permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-oidc-role
          aws-region: us-east-1
          role-duration-seconds: 3600 # 60 minutes; use role_session_duration_minutes * 60

      - run: aws sts get-caller-identity
```

The deployment workflow still uses `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` to deploy or update the CloudFormation stack. Keep these secrets if you plan to use this workflow again; otherwise, remove them and the bootstrap IAM user's access keys after deployment.

---

## Template parameters

| Parameter | Description | Default |
|---|---|---|
| `GitHubOrg` | GitHub org/user | *(required)* |
| `GitHubRepo` | Repo name or `*` | *(required)* |
| `ExistingOIDCProviderArn` | Existing GitHub OIDC provider ARN, or empty to create one | *(empty)* |
| `GitHubSubject` | OIDC subject suffix for refs or environments | `ref:refs/heads/main` |
| `RoleName` | IAM role name | `github-actions-oidc-role` |
| `RoleManagedPolicyArns` | Comma-separated managed policies attached to the role | `arn:aws:iam::aws:policy/AdministratorAccess` |
| `RoleMaxSessionDurationSeconds` | Maximum IAM role session duration in seconds | `3600` |

The role gets **AdministratorAccess by default**, granting full AWS account permissions. This is highly privileged; choose narrower managed policies with `role_managed_policy_arns` whenever possible. When that policy is selected, the workflow targets the protected `administrator-access` environment and pauses for required reviewer approval before deploying.

### Restricting the trust further

Set `github_subject` to restrict which refs or GitHub environments can assume the role. It is the part after `repo:<org>/<repo>:` in the OIDC `sub` claim:

| `github_subject` | Allows |
|---|---|
| `*` | Any ref or environment in the selected repo |
| `ref:refs/heads/main` | The `main` branch |
| `ref:refs/heads/release/*` | Branches under `release/` |
| `ref:refs/tags/v*` | Tags starting with `v` |
| `environment:production` | Jobs using the `production` GitHub environment |

---

## Troubleshooting

| Problem | Cause / Fix |
|---|---|
| `Provider with url https://token.actions.githubusercontent.com already exists` | Set `existing_oidc_provider_arn` to the ARN of the existing provider. Do not delete a provider that other roles or stacks use. |
| `AccessDenied` on `iam:*` or `cloudformation:*` | The deployer IAM user is missing permissions. See the policy in step 2. |
| `Requires capabilities : [CAPABILITY_NAMED_IAM]` | Already handled by the workflow. If deploying manually, add `--capabilities CAPABILITY_NAMED_IAM`. |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | The `sub` condition doesn't match your repo/branch, or `id-token: write` is missing in the calling workflow. |
| Stack in `ROLLBACK_COMPLETE` | Delete the stack and re-run: `aws cloudformation delete-stack --stack-name github-oidc` |

---

## Cleanup

```bash
aws cloudformation delete-stack --stack-name github-oidc --region us-east-1
```

This removes the role and any OIDC provider created by this stack. A provider supplied through `existing_oidc_provider_arn` is external to the stack and is left in place.

---

## Security notes

- The deployment workflow still reads bootstrap AWS credentials from repository secrets. Keep them only if you need to run or update this deployment workflow; otherwise remove the secrets and deactivate the bootstrap keys.
- Scope `GitHubRepo` to a specific repo instead of `*` wherever possible.
- Review IAM permissions attached to the role before using it for deployments.

## License

This project is licensed under the MIT License. See [LICENSE](./LICENSE).
