# GitHub Actions → AWS OIDC Provider (CloudFormation)

Create an **AWS IAM OIDC identity provider for GitHub Actions** (and optionally an IAM role) in your own AWS account, using CloudFormation deployed from a GitHub Actions workflow. Fork or clone it, add two secrets, click **Run workflow**, and get the ARNs back.

Once set up, your other GitHub workflows can authenticate to AWS **without long-lived access keys**.

---

## What you get

| Resource | Description |
|---|---|
| `AWS::IAM::OIDCProvider` | Trusts `token.actions.githubusercontent.com` with audience `sts.amazonaws.com` |
| `AWS::IAM::Role` *(optional)* | A role that only your chosen GitHub org/repo can assume via OIDC |

**Outputs:** `OIDCProviderArn` and `RoleArn` are printed in the job log, shown in the run summary, and exposed as job outputs.

## Repo structure

```
.
├── cfn/
│   └── github-oidc.yaml              # CloudFormation template
├── .github/
│   └── workflows/
│       └── deploy-oidc.yml           # Deployment workflow (manual trigger)
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

You need one-time credentials to create the OIDC provider (chicken-and-egg: OIDC doesn't exist yet). Create an IAM user with programmatic access and attach a policy like the one below.

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
| `github_repo` | Repo allowed to assume the role (`*` = all repos in the org) | `*` |
| `create_role` | Also create the IAM role? | `true` |
| `role_name` | Name of the IAM role | `github-actions-oidc-role` |

### 5. Get your ARNs

Open the completed run. The ARNs are in the **run summary** and in the *Get stack outputs (ARNs)* step:

```
OIDC Provider ARN: arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com
Role ARN:          arn:aws:iam::123456789012:role/github-actions-oidc-role
```

You can also fetch them from the CLI:

```bash
aws cloudformation describe-stacks --stack-name github-oidc \
  --query "Stacks[0].Outputs" --output table
```

---

## Using the role in your other workflows

After deployment you no longer need the IAM user keys. In any workflow in the allowed repo:

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

      - run: aws sts get-caller-identity
```

**Then clean up:** delete the IAM user's access keys (and the user), and remove the two repo secrets.

---

## Template parameters

| Parameter | Description | Default |
|---|---|---|
| `GitHubOrg` | GitHub org/user | *(required)* |
| `GitHubRepo` | Repo name or `*` | `*` |
| `CreateRole` | `true` / `false` | `true` |
| `RoleName` | IAM role name | `github-actions-oidc-role` |
| `RoleManagedPolicyArn` | Managed policy attached to the role | `arn:aws:iam::aws:policy/ReadOnlyAccess` |

The role gets **ReadOnlyAccess by default**. Change `RoleManagedPolicyArn` (or edit the template) to grant what your pipelines actually need, following least privilege.

### Restricting the trust further

The trust policy matches `repo:<org>/<repo>:*`. To lock it down to a branch or environment, edit the `sub` condition in `cfn/github-oidc.yaml`, e.g.:

```yaml
token.actions.githubusercontent.com:sub: !Sub "repo:${GitHubOrg}/${GitHubRepo}:ref:refs/heads/main"
```

---

## Troubleshooting

| Problem | Cause / Fix |
|---|---|
| `Provider with url https://token.actions.githubusercontent.com already exists` | AWS allows **one** OIDC provider per URL per account. Your account already has one. Reuse the existing provider ARN and create only a role, or delete/import the existing provider. |
| `AccessDenied` on `iam:*` or `cloudformation:*` | The deployer IAM user is missing permissions. See the policy in step 2. |
| `Requires capabilities : [CAPABILITY_NAMED_IAM]` | Already handled by the workflow. If deploying manually, add `--capabilities CAPABILITY_NAMED_IAM`. |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | The `sub` condition doesn't match your repo/branch, or `id-token: write` is missing in the calling workflow. |
| Stack in `ROLLBACK_COMPLETE` | Delete the stack and re-run: `aws cloudformation delete-stack --stack-name github-oidc` |

---

## Cleanup

```bash
aws cloudformation delete-stack --stack-name github-oidc --region us-east-1
```

This removes the OIDC provider and the role created by this stack.

---

## Security notes

- Prefer **repository secrets** (or environment secrets with required reviewers) for the bootstrap keys.
- Rotate or delete the bootstrap IAM keys once OIDC is working.
- Scope `GitHubRepo` to a specific repo instead of `*` wherever possible.
- Review IAM permissions attached to the role before using it for deployments.

## License

MIT (or your preferred license). Add a `LICENSE` file before publishing.
