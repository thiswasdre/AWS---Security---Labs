# Lab 06 — flaws.cloud: IAM Enumeration → Lambda → API Gateway (final level)

**Date:** 2026-11-03
**Difficulty:** Intermediate
**Time taken:** ~20 minutes

---

## 🎯 Objective
Complete the final level of flaws.cloud. Starting from a set of limited, read-only
credentials, enumerate IAM policies, discover a Lambda function, read its resource
policy to find the API Gateway that triggers it, and call that endpoint to retrieve
the final secret.

## 🧰 AWS services / tools used
- IAM (the `SecurityAudit` managed policy + a custom policy)
- AWS Lambda
- Amazon API Gateway
- AWS CLI (v2)

## 🗺️ What I did (steps)

1. **Loaded the provided `Level6` user credentials** as a CLI profile and confirmed
   the identity:
   ```
   aws configure --profile level6
   aws sts get-caller-identity --profile level6
   ```
2. **Listed the user's attached policies** to see what it could do:
   ```
   aws iam list-attached-user-policies --user-name Level6 --profile level6
   ```
   Found `MySecurityAudit` (broad read-only auditing) and `list_apigateways` (a custom
   policy — a strong hint about where to look).
3. **Enumerated Lambda functions** and found one named `Level6`:
   ```
   aws lambda list-functions --profile level6 --region us-west-2
   ```
4. **Read the Lambda's resource policy** to see what's allowed to invoke it:
   ```
   aws lambda get-policy --function-name Level6 --profile level6 --region us-west-2
   ```
   The policy's `SourceArn` revealed the API Gateway ID and path:
   ```
   arn:aws:execute-api:us-west-2:975426262029:s33ppypa75/*/GET/level6
   ```
5. **Looked up the API Gateway stage name:**
   ```
   aws apigateway get-stages --rest-api-id s33ppypa75 --profile level6 --region us-west-2
   ```
   Stage: `Prod`.
6. **Built the public URL and visited it**, triggering the Lambda and completing the
   challenge:
   ```
   https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6
   ```

## 💡 What I learned
- **Read-only "audit" access is powerful for reconnaissance.** With no write
  permissions at all, I mapped out a Lambda function, its role, and the API Gateway
  wired to it — everything needed to find the hidden endpoint.
- A **Lambda's resource policy** tells you exactly which API Gateway (and path) can
  invoke it — enough to reconstruct the public URL.
- Over-broad read access plus predictable service wiring lets an attacker **chain
  services together** to reach data, without ever needing write access.

## 🛡️ How it maps to the real world
- Scope even **read-only / `SecurityAudit`** access carefully — reconnaissance is the
  first stage of every real attack.
- **Lock down who can invoke Lambda functions**, and don't rely on an API Gateway URL
  being secret — obscurity is not security (the recurring theme of this whole
  challenge).
- **Monitor** API Gateway and Lambda access logs, and apply least privilege to both
  human users and service roles.

## 🧹 Cleanup
- No resources created (read-only enumeration). Removed the temporary `level6` CLI
  profile afterward.

---
## 🏁 Challenge complete
This was the final level — **all six levels of flaws.cloud completed.** The series
walked through the core of AWS attack surface: public S3, authenticated-user
misconfigurations, secrets in git history, exposed EBS snapshots, SSRF against the
EC2 metadata service, and IAM/Lambda/API-Gateway enumeration.

*Part of my AWS cloud security lab portfolio.*
