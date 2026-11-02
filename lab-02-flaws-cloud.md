# Lab 02 — flaws.cloud: The "Authenticated Users" Trap

**Date:** 2026-11-01
**Difficulty:** Beginner
**Time taken:** ~30 minutes

---

## 🎯 Objective
Work through Level 2 of flaws.cloud, where an S3 bucket blocks anonymous access
but is misconfigured to allow **any authenticated AWS user**. The goal: show that
"Authenticated Users" does not mean "my users" — it means anyone on Earth with an
AWS account.

## 🧰 AWS services / tools used
- Amazon S3
- AWS CLI (v2)
- IAM (my own account's credentials)

## 🗺️ What I did (steps)

1. **Tried the Level 1 trick first — anonymous, no credentials.** This time it was
   blocked:
   ```
   aws s3 ls s3://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/ --no-sign-request --region us-west-2
   ```
   Result:
   ```
   An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
   ```
   So this bucket is locked down against total strangers — more secure than Level 1.

2. **Tried again, signed in as myself** (removed `--no-sign-request`, so the CLI used
   my own AWS credentials):
   ```
   aws s3 ls s3://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/ --region us-west-2
   ```
   This time it worked and listed the contents:
   ```
   everyone.png
   hint1.html
   hint2.html
   index.html
   robots.txt
   secret-e4443fc.html      <-- the file that completes the level
   ```

3. **Opened the exposed file** in a browser to confirm the solve:
   ```
   http://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/secret-e4443fc.html
   ```

## 💡 What I learned
- The **only** difference between failure and success was being logged into *any*
  AWS account. I was never given specific permission — I just qualified as "an
  authenticated AWS user."
- In S3, the **"Authenticated Users" grant means every AWS account in the world**,
  not just people inside your organization. It's a notorious misconfiguration
  because the name is so misleading.
- Locking a bucket against anonymous access is **not enough** — the grant still has
  to be scoped to specific principals.

## 🛡️ How it maps to the real world
This mistake has caused real breaches, because teams assume "Authenticated Users"
is restrictive when it's nearly as open as public. A cloud security engineer
prevents it by: never using the "Authenticated Users" grant, scoping bucket
policies to specific IAM principals or accounts, enabling **S3 Block Public
Access**, and auditing bucket ACLs/policies with tools like **AWS Config** or
**Prowler**.

## 🧹 Cleanup
No resources created — flaws.cloud is externally hosted, so there was nothing to
tear down.

---
*Part of my AWS cloud security lab portfolio.*
