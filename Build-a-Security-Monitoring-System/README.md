# Build a Security Monitoring System

## Overview

In this project, I built a full security monitoring and alerting pipeline on AWS: storing a secret in Secrets Manager, tracking access to it with CloudTrail, routing those logs into CloudWatch, and triggering an automated email alert via SNS whenever the secret was accessed.

The most valuable part of this project wasn't the initial setup — it was the troubleshooting. My first attempt at the alerting pipeline didn't work, and diagnosing exactly where it broke down (and why) taught me more about how these services fit together than a clean, error-free run would have.

## Services Used

- **AWS Secrets Manager** — stored the secret being monitored
- **AWS CloudTrail** — recorded management events across the account, including secret access
- **Amazon CloudWatch (Logs, Metric Filters, Alarms)** — ingested CloudTrail logs, tracked access via a custom metric, and triggered alerting
- **Amazon SNS** — delivered email notifications when the alarm triggered
- **AWS CLI / CloudShell** — used as an alternate method to access the secret and generate test events

## What I Did

### 1. Created a Secret to Monitor
Stored a dummy secret (`TopSecretInfo`) in Secrets Manager — the resource this entire monitoring pipeline is built to watch.

![Secret Created in Secrets Manager](./Screen%20Shot%202026-08-16%20at%204.05.16%20PM.png)

### 2. Configured CloudTrail
Created a multi-region trail (`secrets-manager-trail`) to record management events, specifically scoped to catch both Read and Write API activity, with KMS and RDS Data API events excluded to cut down on noise.

![CloudTrail Trail Successfully Created](./Screen%20Shot%202026-08-16%20at%204.29.45%20PM.png)

### 3. Verified CloudTrail Captured Secret Access
Retrieved the secret's value through the console and confirmed CloudTrail logged a `GetSecretValue` event in Event history, matching the exact timestamp of the access.

![CloudTrail Event Confirming GetSecretValue Was Recorded](./Screen%20Shot%202026-08-16%20at%204.43.06%20PM.png)

### 4. Routed Logs to CloudWatch and Built a Metric Filter
Enabled CloudWatch Logs on the trail, creating a new log group (`nextwork-secretsmanager-loggroup`) and confirmed the trail's configuration correctly pointed to it.

![Trail Details Confirming CloudWatch Logs Enabled](./Screen%20Shot%202026-08-17%20at%206.56.25%20PM.png)

Then confirmed CloudTrail logs were actually flowing into the log group.

![CloudWatch Log Group Details](./Screen%20Shot%202026-08-17%20at%207.01.03%20PM.png)
![Log Events Populating in the Log Group](./Screen%20Shot%202026-08-17%20at%207.39.00%20PM.png)

Then built a metric filter (`SecurityMetrics / Secret is accessed`) intended to increment by 1 every time a `GetSecretValue` event appeared in the logs.

![Reviewing the Metric Filter Configuration](./Screen%20Shot%202026-08-17%20at%207.56.44%20PM.png)

### 5. Set Up the CloudWatch Alarm and SNS Alert
Created a CloudWatch Alarm on the metric (threshold: >= 1 within a 5-minute period) and a new SNS topic (`SecurityAlarms`) to deliver an email notification when the alarm triggered.

![Alarm Created, SNS Subscription Pending Confirmation](./Screen%20Shot%202026-08-17%20at%208.22.57%20PM.png)
![Email Subscription Confirmed](./Screen%20Shot%202026-08-17%20at%208.25.20%20PM.png)

### 6. Tested the Pipeline — and Troubleshot It
Retrieved the secret again to trigger the alarm end-to-end. No email arrived. Rather than assume something was fundamentally broken, I traced the pipeline stage by stage:

1. **CloudTrail** — confirmed a fresh `GetSecretValue` event was recorded at the correct timestamp. ✅ Working.
2. **Log delivery** — confirmed the event reached the CloudWatch log group, with a new log stream entry matching the access time. ✅ Working.
3. **Metric filter** — graphed the `Secret is accessed` metric directly instead of assuming it was incrementing correctly, and found it was flat, barely moving above 0 even after multiple secret accesses. ❌ This was the break.

![Metric Graph Showing the Filter Was Not Incrementing Correctly](./Screen%20Shot%202026-08-17%20at%209.22.46%20PM.png)

**Root cause:** my metric filter pattern was a plain text match (`"GetSecretValue"`), which searches for that string anywhere in the raw log line rather than matching the specific `eventName` field in the structured JSON. This caused unreliable matching — it even matched an unrelated `DescribeMetricFilters` API call that happened to mention "GetSecretValue" in its request parameters, while inconsistently catching the actual secret-access events I cared about.

**Fix:** updated the filter pattern to target the JSON field explicitly:
```
{ $.eventName = "GetSecretValue" }
```

After saving the corrected filter and retrieving the secret again, the metric incremented correctly, the alarm transitioned into the **In alarm** state, and the SNS email notification arrived as expected — confirming the full pipeline worked end-to-end.

## Key Takeaways

- **A working configuration on paper isn't the same as a working system.** Every individual component (CloudTrail, the log group, the metric filter, the alarm, the SNS subscription) looked correctly set up, but the pipeline still failed silently until I validated each stage's actual output, not just its configuration.
- **CloudWatch Logs metric filters need JSON-aware syntax for structured logs.** A plain text pattern like `"GetSecretValue"` performs a substring search across the entire raw log line, which can produce false positives (matching unrelated events that happen to mention the term) and false negatives (missing the actual field you're trying to match). Using `{ $.fieldName = "value" }` syntax targets the specific JSON field and is far more reliable for CloudTrail-style structured logs.
- **Systematic, stage-by-stage troubleshooting beats guessing.** Instead of randomly changing settings, I isolated each link in the chain (CloudTrail → CloudWatch Logs → metric filter → alarm → SNS) and confirmed or ruled out each one before moving to the next, which is what actually surfaced the real root cause efficiently.
- **CloudTrail treats secret retrieval as a "Write" management event**, not a Read or Data event, specifically to ensure it's captured even in cost-conscious environments that might otherwise skip logging read-only or data-plane activity.
- **SNS subscriptions require explicit confirmation** before any notifications will actually deliver — an easy step to overlook, but a hard blocker if missed.
- This project reflects a very real pattern in cloud security work: building the monitoring infrastructure is often the easy part; verifying it actually works as intended, and diagnosing it when it doesn't, is where the real skill is required.
