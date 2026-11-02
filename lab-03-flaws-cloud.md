# Lab 03 — flaws.cloud: Secrets in Git History & Leaked Credentials

**Date:** 2026-11-02
**Difficulty:** Intermediate
**Time taken:** ~45 minutes

---

## 🎯 Objective
Work through Level 3 of flaws.cloud, where a public S3 bucket exposed a `.git`
directory. Recover AWS access keys that were committed and later "deleted," then
use them to pivot into the account — demonstrating why secrets in git history and
leaked long-lived keys are so dangerous.

## 🧰 AWS services / tools used
- Amazon S3
- AWS CLI (v2) — including named profiles
- git (version-history forensics)
- IAM access keys

## 🗺️ What I did (steps)

1. **Listed the bucket anonymously** and noticed a folder that should never be
   public — a `.git` directory:
   ```
   aws s3 ls s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud --no-sign-request --region us-west-2
   ```
   ```
   PRE .git/          <-- the whole source-code history, exposed
   ...
   ```
2. **Downloaded the entire bucket**, including the `.git` folder:
   ```
   mkdir ~/level3 && cd ~/level3
   aws s3 sync s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/ . --no-sign-request --region us-west-2
   ```
3. **Read the commit history:**
   ```
   git log
   ```
   Two commits — the newest one was a giveaway:
   ```
   "Oops, accidentally added something I shouldn't have"   (latest)
   "first commit"
   ```
4. **Realised the secret was removed in the latest commit but still lived in the
   first commit**, and checked out that earlier version:
   ```
   git checkout <first-commit-hash>
   ls
   cat access_keys.txt
   ```
   This revealed an AWS **Access Key ID** and **Secret Access Key** (redacted here —
   see note below).
5. **Loaded the stolen keys as a separate CLI profile** and acted as that account:
   ```
   aws configure --profile flaws
   aws s3 ls --profile flaws
   ```
6. The account's bucket list revealed the **Level 4 bucket**, confirming the solve:
   ```
   level4-xxxxxxxxxxxxxxxx.flaws.cloud
   ```

## 💡 What I learned
- An exposed `.git` directory hands an attacker the **entire source history** of a
  project.
- **Git never forgets.** A secret that's committed and then "removed" in a later
  commit is still fully recoverable with `git log` + `git checkout`. Deleting it
  does not delete it.
- A **leaked long-lived access key** gives an attacker everything that identity can
  do — this is one of the most common real-world cloud breach patterns.
- Pivoting with stolen credentials is as simple as loading them into a named CLI
  profile.

## 🛡️ How it maps to the real world
Developers accidentally committing AWS keys is behind a huge share of real cloud
breaches. A cloud security engineer prevents this by:
- **Never committing secrets** — use `git-secrets` / pre-commit hooks, AWS Secrets
  Manager, or environment variables.
- **Never deploying a `.git` folder** to a web server or S3 bucket.
- **Using short-lived credentials** (IAM roles / STS) instead of long-lived access
  keys.
- **Scanning & rotating** — tools like truffleHog and GitHub secret scanning catch
  leaks; AWS itself detects exposed keys and can auto-quarantine them.

## 🔒 Note on the redacted keys
The recovered Access Key ID and Secret Key are **intentionally redacted** in this
writeup. Even though these are flaws.cloud's deliberately-exposed practice
credentials, publishing real-looking keys in a public portfolio is a bad habit —
so I'm modelling the correct behaviour: **credentials never go in a repo, not even
in a lab writeup.**

## 🧹 Cleanup
- Lab files downloaded to `~/level3` (local only).
- Removed the temporary `flaws` CLI profile afterward so no stray credentials
  remain on my machine.

---
*Part of my AWS cloud security lab portfolio.*
