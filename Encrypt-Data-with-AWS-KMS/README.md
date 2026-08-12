# Encrypt Data with AWS KMS

Hands-on project using AWS Key Management Service (KMS) to encrypt a DynamoDB table at rest, then validating the encryption boundary by testing access with a scoped-permission IAM user.

## Key Concepts

- AWS Key Management Service (KMS)
- Customer Managed Keys (CMK) vs. AWS Managed Keys
- Symmetric encryption
- Amazon DynamoDB
- Encryption at rest
- Key policies and key administrators/users
- Transparent data encryption
- IAM permission scoping (`kms:Decrypt`)

## Project Steps

### Step 1: Create a KMS Key

Created a customer managed key (`nextwork-kms-key`) in AWS KMS, configured for symmetric encrypt/decrypt operations, with an IAM Admin user set as both key administrator and key user.

![KMS key created showing alias](./Screen%20Shot%202026-08-12%20at%203.58.37%20PM.png)

### Step 2: Create and Encrypt a DynamoDB Table

Created a DynamoDB table (`nextwork-kms-table`) with partition key `id`, then configured its encryption settings to use the customer managed KMS key instead of a default AWS-owned or AWS-managed key.

![Manage encryption settings showing customer managed key selected](./Screen%20Shot%202026-08-12%20at%204.06.32%20PM.png)

### Step 3: Add Data to the Table

Added a test item (`id: 1`) to the encrypted table and confirmed it was readable as an authorized user, demonstrating transparent data encryption — DynamoDB automatically decrypts data on behalf of users with the correct KMS permissions.

![Item added to DynamoDB table](./Screen%20Shot%202026-08-12%20at%204.09.20%20PM.png)

### Step 4: Create a Test User

Created a scoped IAM user (`nextwork-kms-user`) with full DynamoDB access but no permissions on the KMS key, to test whether encryption actually blocks unauthorized access to the underlying data.

![Test user created, console sign-in details generated](./Screen%20Shot%202026-08-12%20at%204.17.33%20PM.png)

### Step 5: Validate KMS Encryption

Logged in as the test user in an incognito window and attempted to view items in the encrypted table.

Result: **Access Denied** (`kms:Decrypt` not permitted) — confirming that DynamoDB access alone is not sufficient to read encrypted data; the KMS key permission is a separate, enforced security boundary.

![Access denied error when viewing encrypted table as test user](./Screen%20Shot%202026-08-12%20at%204.20.35%20PM.png)

## Key Takeaways

- Encryption at rest and resource-level access (e.g., DynamoDB table permissions) are separate security boundaries — a user can have full access to a database service and still be blocked from reading data if they lack the corresponding KMS key permissions.
- Customer managed keys (CMKs) provide full control over key policies, administrators, and users, unlike AWS-owned or AWS-managed keys where AWS controls access.
- Transparent data encryption means authorized users interact with decrypted data seamlessly, while the underlying storage remains encrypted — security without added friction for legitimate access.
- Testing security controls by attempting the blocked action as a genuinely restricted user (rather than just trusting the policy configuration) is the only way to confirm access controls work as intended.
