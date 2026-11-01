# Lab 01 — flaws.cloud: Finding an Exposed S3 Bucket

**Date:** 2026-10-30
**Difficulty:** Beginner
**Time taken:** ~1 hour (including first-time AWS CLI setup)

---

## 🎯 Objective
Set up the AWS CLI on my Mac, then work through Level 1 of the flaws.cloud
challenge — a deliberately vulnerable AWS environment — to find and understand
a misconfigured S3 bucket that exposes its contents without any credentials.

## 🧰 AWS services / tools used
- Amazon S3
- AWS CLI (v2)
- IAM (access keys for CLI authentication)

## 🗺️ What I did (steps)

1. **Connected my Mac to AWS.** Installed the AWS CLI, then configured it with
   an IAM user's access keys:
   ```
   aws configure
   ```
2. **Verified the connection** — confirmed the CLI was authenticated:
   ```
   aws sts get-caller-identity
   ```
   (Returned my account ID and IAM user ARN — login working.)
3. **Listed the target bucket with NO credentials** — the core of the lab:
   ```
   aws s3 ls s3://flaws.cloud/ --no-sign-request --region us-west-2
   ```
   The `--no-sign-request` flag means "try this with no authentication at all."
   The bucket responded with its full file list:
   ```
   hint1.html
   hint2.html
   hint3.html
   index.html
   logo.png
   robots.txt
   secret-dd02c7c.html      <-- a file meant to be hidden
   ```
4. **Opened the exposed file** in a browser to complete the level:
   ```
   http://flaws.cloud/secret-dd02c7c.html
   ```

## 💡 What I learned
- An S3 bucket with public **list** permissions leaks every file name inside it
  — including files the owner assumed were "hidden" just because nothing linked
  to them.
- `--no-sign-request` is a quick way to test whether a bucket responds to
  **anonymous** (unauthenticated) requests — a basic but important recon check.
- "Security through obscurity" (hiding instead of locking) is not real security.
  A secret URL is still public if anyone can enumerate it.

## 🛡️ How it maps to the real world
Publicly exposed S3 buckets are one of the most common causes of real-world AWS
data breaches. The fix is straightforward and something a cloud security
engineer checks for constantly: enable **S3 Block Public Access**, apply
least-privilege bucket policies, and enable **GuardDuty** to alert on anonymous
access patterns.

## 🧹 Cleanup
No resources created — flaws.cloud is externally hosted, so there was nothing to
tear down. (For account-based labs, this is where I'd record `terraform destroy`
or manual deletion.)

---
*Part of my AWS cloud security lab portfolio.*
