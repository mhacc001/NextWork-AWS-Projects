# Cloud Security with AWS IAM

Hands-on project using AWS Identity and Access Management (IAM) to control access to AWS resources. Simulates onboarding a new intern at a fictional company (NextWork) by launching EC2 instances for production and development environments, then using IAM policies, groups, and users to restrict the intern's access to development resources only.

## Key Concepts

- AWS IAM (Identity and Access Management)
- Amazon EC2 (Elastic Compute Cloud)
- IAM Policies (JSON-based permission rules)
- IAM User Groups
- Resource tagging for environment-based access control
- Account Aliases
- IAM Policy Simulator

## Project Steps

### Step 1: Launch EC2 Instances

Launched two EC2 instances, one tagged for production and one for development, to simulate increased compute capacity for the holiday season.

- Instance 1: `nextwork-prod-[name]` — tagged `Env: production`
- Instance 2: `nextwork-dev-[name]` — tagged `Env: development`

![EC2 instances running](./Screen_Shot_2026-08-12_at_1_50_05_PM.png)

### Step 2: Create an IAM Policy

Created a custom IAM policy (`NextWorkDevEnvironmentPolicy`) restricting access to EC2 instances tagged `Env: development`, while explicitly denying the ability to create or delete tags — preventing an intern from disguising a production instance as a development one.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Env": "development"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:DeleteTags",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

![IAM policy JSON](./Screen_Shot_2026-08-12_at_2_06_04_PM.png)

### Step 3: Create an AWS Account Alias

Created a friendly account alias (`nextwork-alias-[name]`) to simplify the sign-in URL for new IAM users, replacing the default numeric account ID in the console login link.

![Account alias setup](./Screen_Shot_2026-08-12_at_2_15_06_PM.png)

### Step 4: Create IAM Users and User Groups

Created an IAM user group (`nextwork-dev-group`) with the `NextWorkDevEnvironmentPolicy` attached, then created an IAM user for the intern and added them to the group — centralizing permission management instead of applying policies to individual users.

![IAM user and group creation](./Screen_Shot_2026-08-12_at_2_20_29_PM.png)

### Step 5: Test the Intern's Access

Logged in as the new IAM user to verify permission boundaries:

- Attempted to stop the **production** instance → **Access Denied** (expected — not authorized under the attached policy)
- Attempted to stop the **development** instance → **Success**

![Access denied on production instance](./Screen_Shot_2026-08-12_at_2_34_11_PM.png)
![Successful stop on development instance](./Screen_Shot_2026-08-12_at_2_35_39_PM.png)

## Key Takeaways

- IAM policies use `Effect`, `Action`, `Resource`, and optional `Condition` blocks to define precisely what a user can and cannot do, and under what circumstances.
- `Deny` always takes precedence over `Allow` within a policy evaluation, which is critical for building safe least-privilege access controls.
- Resource tagging (e.g., `Env: production` vs `Env: development`) combined with IAM policy conditions is an effective way to scope access without creating a separate policy per resource.
- IAM user groups centralize permission management — attaching a policy to a group rather than individual users makes onboarding and offboarding far more scalable.
- The IAM Policy Simulator is a faster way to validate permission logic than manually logging in as each user to test access boundaries.
