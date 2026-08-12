# Cloud Security with AWS IAM

Hands-on project using AWS Identity and Access Management (IAM) to control access to AWS resources. Simulates onboarding a new intern at a fictional company (NextWork) by launching EC2 instances for production and development environments, then using IAM policies, groups, and users to restrict the intern's access to development resources only.

## Key Concepts

- AWS IAM (Identity and Access Management)
- Amazon EC2 (Elastic Compute Cloud)
- IAM Policies (JSON-based permission rules)
- IAM User Groups
- Resource tagging for environment-based access control
- Account Aliases

## Project Steps

### Step 1: Launch EC2 Instances

Launched two EC2 instances, one tagged for production and one for development, to simulate increased compute capacity for the holiday season.

- Instance 1: `nextwork-prod-Marie` — tagged `Env: production`
- Instance 2: `nextwork-dev-Marie` — tagged `Env: development`

![EC2 instances correctly named and tagged](./Screen%20Shot%202026-08-12%20at%201.57.25%20PM.png)

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

![IAM policy JSON created](./Screen%20Shot%202026-08-12%20at%202.06.04%20PM.png)

### Step 3: Create an AWS Account Alias

Created a friendly account alias (`nextwork-alias-marie`) to simplify the sign-in URL for new IAM users, replacing the default numeric account ID in the console login link.

![Account alias setup](./Screen%20Shot%202026-08-12%20at%202.15.06%20PM.png)

### Step 4: Create IAM Users and User Groups

Created an IAM user group (`nextwork-dev-group`) with the `NextWorkDevEnvironmentPolicy` attached, then created an IAM user (`nextwork-dev-marie`) for the intern and added them to the group — centralizing permission management instead of applying policies to individual users.

![New user created, console sign-in details generated](./Screen%20Shot%202026-08-12%20at%202.20.29%20PM.png)

![IAM users list confirming group membership](./Screen%20Shot%202026-08-12%20at%202.23.31%20PM.png)

### Step 5: Test the Intern's Access

Logged in as the new IAM user in an incognito window to verify permission boundaries.

**Signing in as the intern:**

![IAM user sign-in form](./Screen%20Shot%202026-08-12%20at%202.32.14%20PM.png)

**Fresh console view as a new user** — as expected, several dashboard panels show no history or access yet, since this identity has never been used before:

![New intern's console home view](./Screen%20Shot%202026-08-12%20at%202.32.45%20PM.png)

**Attempting to stop the production instance:**

![Stop instance dialog on production](./Screen%20Shot%202026-08-12%20at%202.33.58%20PM.png)

Result: **Access Denied** — confirming the policy correctly blocks the intern from managing production resources.

![Access denied error on production instance](./Screen%20Shot%202026-08-12%20at%202.34.11%20PM.png)

**Attempting to stop the development instance:**

Result: **Success** — the intern's scoped permissions correctly allow managing development resources.

![Successful stop on development instance](./Screen%20Shot%202026-08-12%20at%202.35.39%20PM.png)

## Key Takeaways

- IAM policies use `Effect`, `Action`, `Resource`, and optional `Condition` blocks to define precisely what a user can and cannot do, and under what circumstances.
- `Deny` always takes precedence over `Allow` within a policy evaluation, which is critical for building safe least-privilege access controls.
- Resource tagging (e.g., `Env: production` vs `Env: development`) combined with IAM policy conditions is an effective way to scope access without creating a separate policy per resource.
- IAM user groups centralize permission management — attaching a policy to a group rather than individual users makes onboarding and offboarding far more scalable.
- Testing IAM policies by actually logging in as the scoped user (rather than just trusting the policy JSON) confirms real-world behavior and catches misconfigurations early.
