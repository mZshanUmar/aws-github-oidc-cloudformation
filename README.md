# GitHub Actions → AWS OIDC Provider + Role (CloudFormation)

This project uses CloudFormation and GitHub Actions to create an AWS IAM role trusted by GitHub Actions. You can create a GitHub OIDC provider or reuse one already in your account. Workflows using the role can access AWS without long-lived AWS keys.

---

## What you get

| Resource | Description |
|---|---|
| `AWS::IAM::OIDCProvider` | Created unless you provide an existing provider ARN |
| `AWS::IAM::Role` | Trusted by the GitHub org, repository, and subject you configure |

The workflow shows redacted ARNs in the run summary; full ARNs are available in the CloudFormation stack Outputs.

## OIDC and credential lifetime

The role's maximum session duration is set to the IAM minimum of 60 minutes in the CloudFormation template. To request a longer session, first raise `MaxSessionDuration` in `cfn/github-oidc.yaml` (valid range: 3600–43200 seconds), then set `role-duration-seconds` in each workflow that assumes the role. For example, to request 90 minutes, set the role maximum to at least 5400 seconds and use `role-duration-seconds: 5400`. The GitHub OIDC token is valid for 5 minutes and is separate from the AWS session.

---

## Quick start

### 1. Fork or clone this repo

Fork the repository, or clone it and push it to your own GitHub repository:

```bash
git clone https://github.com/mZshanUmar/aws-github-oidc-cloudformation.git
```

Enable GitHub Actions in your repository (**Settings → Actions → General**).

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
        "iam:UpdateRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:ListAttachedRolePolicies",
        "iam:ListRolePolicies"
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

Never commit these keys or pass them as workflow inputs. Inputs are visible in plain text.

### 4. Run the workflow

**Actions → Deploy GitHub OIDC Provider → Run workflow**, then fill in:

| Input | Description | Default |
|---|---|---|
| `stack_name` | CloudFormation stack name | `github-oidc` |
| `oidc_provider_arn` | Existing GitHub OIDC provider ARN; leave blank to create one | *(empty)* |
| `role_name` | IAM role name | `GitHubOIDCRole` |
| `aws_managed_policy_arns` | Comma-separated managed policy ARNs to attach to the role | `arn:aws:iam::aws:policy/AdministratorAccess` |
| `github_org` | Your GitHub org or username | *(required)* |
| `github_repo` | Repo allowed to assume the role; `*` explicitly allows all repos in the org | *(required)* |
| `github_subject` | OIDC subject suffix to restrict refs or environments | `ref:refs/heads/main` |

#### Already have a GitHub OIDC provider in your AWS account?

Reuse the provider, set `oidc_provider_arn` to a GitHub provider ARN in the same AWS account, configured with `https://token.actions.githubusercontent.com` and audience `sts.amazonaws.com`. The stack does not manage a supplied provider or change other roles' trust policies.

List provider ARNs with `aws iam list-open-id-connect-providers --query "OpenIDConnectProviderList[].Arn" --output text`.

#### Restricting the trust

`github_subject` is the part after `repo:<org>/<repo>:` in the OIDC `sub` claim. Use it to restrict which refs or GitHub environments can assume the role:

| `github_subject` | Allows |
|---|---|
| `*` | Any subject in the selected repo, including refs, environments, and pull requests |
| `ref:refs/heads/main` | The `main` branch |
| `ref:refs/heads/release/*` | Branches under `release/` |
| `ref:refs/tags/v*` | Tags starting with `v` |
| `environment:production` | Jobs using the `production` GitHub environment |

#### ⚠️ Administrator Access Policy

`AdministratorAccess` grants full AWS account permissions. When its ARN is selected, the workflow adds a warning to the run summary. This is an informational check for the selected policy, not an analysis of other policies' permissions. Choose narrower policies whenever possible.

Enter multiple managed policy ARNs separated by commas (up to 10 by default), for example:

```text
arn:aws:iam::aws:policy/AmazonEC2FullAccess,arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

Policy ARNs can refer to AWS-managed or existing customer-managed policies.

### 5. Get your ARNs

The run summary redacts the account ID, so use the full values from **CloudFormation → Stacks → your stack → Outputs** or the AWS CLI:

```bash
aws cloudformation describe-stacks --stack-name github-oidc \
  --region us-east-1 \
  --query "Stacks[0].Outputs" --output table
```

Replace the stack name if needed. ARNs identify resources but are not credentials; keep the role's trust and permissions restricted.

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
          role-to-assume: arn:aws:iam::123456789012:role/GitHubOIDCRole
          aws-region: us-east-1
          role-duration-seconds: 3600 # 60 minutes; action input is in seconds

      - run: aws sts get-caller-identity
```

The deployment workflow uses the bootstrap secrets to deploy or update the stack in `us-east-1`. Remove the secrets and deactivate the bootstrap keys when you no longer need them.

---

## Direct CloudFormation deployments

The workflow maps its inputs to the template parameters. For direct deployments, provide all template parameters except `ExistingOidcProviderArn`, which defaults to an empty string. You can always set it to a GitHub provider ARN to reuse one. The role's maximum session duration is set to 3600 seconds in the template; edit `MaxSessionDuration` there to change it.

---

## Troubleshooting

| Problem | Cause / Fix |
|---|---|
| `Provider with url https://token.actions.githubusercontent.com already exists` | Set `oidc_provider_arn` to the ARN of the existing provider. Do not delete a provider that other roles or stacks use. |
| `AccessDenied` on `iam:*` or `cloudformation:*` | The deployer IAM user is missing permissions. See the policy in step 2. |
| `Requires capabilities : [CAPABILITY_NAMED_IAM]` | Already handled by the workflow. If deploying manually, add `--capabilities CAPABILITY_NAMED_IAM`. |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | The `sub` condition doesn't match your repo/branch, or `id-token: write` is missing in the calling workflow. |
| Stack in `ROLLBACK_COMPLETE` | Delete the stack and re-run: `aws cloudformation delete-stack --stack-name github-oidc` |

---

## Cleanup

To delete the stack in GitHub, run **Actions → Destroy GitHub OIDC Stack**, choose the stack name, and enter the same name in the confirmation field. The workflow deletes the stack and waits for deletion to complete.

You can also delete it with the AWS CLI:

```bash
aws cloudformation delete-stack --stack-name github-oidc --region us-east-1
```

This removes the role and any OIDC provider created by the stack. A provider supplied through `oidc_provider_arn` is left in place. Before deleting a stack-owned provider, ensure no other roles or stacks depend on it.

## License

This project is licensed under the MIT License. See [LICENSE](./LICENSE).
