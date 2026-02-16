# OpenClaw VM Installation Plan
Host: Mac mini M1 (Apple Silicon)
Goal: Secure, sandboxed agent runtime for Project Brain automation
Version: v1.0

---

# 1. Architecture Overview

Mac mini (Host)
  ├── BrainArchive/               (Canonical raw storage)
  ├── brain-repo/                 (GitHub clone)
  └── UTM Virtual Machine
        └── Linux (Ubuntu ARM64)
              └── OpenClaw runtime

Security model:
- Agent runs inside Linux VM
- VM has access only to:
    - BrainArchive/agent_inbox
    - BrainArchive/raw
    - brain-repo clone
- No full host disk access
- No access to iCloud root
- No access to other user directories

---

# 2. Required Components

On macOS:
- UTM (Apple Silicon compatible)
- Git
- Xcode Command Line Tools
- iCloud Drive enabled (for BrainInbox sync)

Inside VM:
- Ubuntu Server ARM64
- Python 3.11+
- Git
- OpenSSH
- OpenClaw runtime
- Hashing utilities (sha256sum)
- Optional: Docker (if containerizing OpenClaw)

---

# 3. Install UTM (Virtual Machine Manager)

1. Download UTM (Apple Silicon version)
2. Install in Applications
3. Grant necessary virtualization permissions
4. Reboot if required

---

# 4. Create Linux VM

## 4.1 VM Configuration

OS: Ubuntu Server ARM64 (LTS)
Memory: 4–8 GB
CPU: 4 cores
Disk: 40–80 GB (expandable)
Networking: Shared (NAT)
No bridged networking unless required

## 4.2 Install Ubuntu

- Download ARM64 ISO
- Create new VM in UTM
- Boot installer
- Create non-root user: openclaw
- Enable SSH during install
- Disable password login later (use key auth)

---

# 5. Harden Linux VM

Inside VM:

## 5.1 System Update
sudo apt update && sudo apt upgrade -y

## 5.2 Create Dedicated Agent User
sudo adduser agent
sudo usermod -aG sudo agent

## 5.3 Disable Root SSH
Edit /etc/ssh/sshd_config:
  PermitRootLogin no
  PasswordAuthentication no
Restart SSH.

## 5.4 Enable Firewall
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing

---

# 6. Share Host Folders to VM

In UTM:

Add Shared Directories:

Host Path:
~/BrainArchive/agent_inbox

Mount in VM as:
 /mnt/agent_inbox

Add:
~/BrainArchive/raw

Mount as:
 /mnt/raw

Add:
~/brain-repo

Mount as:
 /mnt/brain-repo

Ensure:
- Read/Write access only where needed
- Avoid sharing entire home directory

---

# 7. Install OpenClaw Runtime

Inside VM:

## 7.1 Install Dependencies
sudo apt install git python3 python3-venv python3-pip -y

## 7.2 Clone OpenClaw
cd /opt
sudo git clone <OpenClaw repository>
sudo chown -R agent:agent openclaw

## 7.3 Create Virtual Environment
cd openclaw
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

---

# 8. GitHub Integration

## 8.1 Create Bot Account
- Dedicated GitHub bot user
- Access limited to brain-repo only

## 8.2 Generate SSH Key Inside VM
ssh-keygen -t ed25519

Add public key to bot account.

## 8.3 Clone Repo in VM
cd /home/agent
git clone git@github.com:<org>/brain-repo.git

Never store personal credentials inside VM.

---

# 9. Configure Agent Permissions

Agent must:

Allowed:
- Read /mnt/agent_inbox
- Read /mnt/raw
- Read/Write brain-repo clone
- Open PR via GitHub API

Forbidden:
- Access outside mounted directories
- Scan full host filesystem
- Direct push to main branch
- Merge PRs

---

# 10. Media Hash Ingestion Rules

When file detected in /mnt/agent_inbox:

1. Compute SHA-256
2. Determine CAS path:
   /mnt/raw/media_store/ab/cd/<hash>.<ext>
3. If file exists, skip duplicate
4. Move file to CAS
5. Generate metadata yaml in repo
6. Update indexes/media_index.yaml
7. Open PR

Never scan entire /mnt/raw for “related” files.

---

# 11. Gmail Ingestion (Optional)

If enabled:

- OAuth scope: gmail.readonly
- Only ingest messages with label VIBE/INGEST
- Optional sender whitelist
- Store raw email in /mnt/raw/email_store
- Commit metadata only

No gmail.modify scope unless strictly required.

---

# 12. Automation Mode

Recommended: Manual Trigger Mode (Phase 1)

Agent runs:
python run_ingestion.py

Later:
- Add cron job for periodic scanning
- Or webhook trigger

Do NOT enable auto-merge.

---

# 13. Logging

Inside VM:

/var/log/openclaw/
  ingestion.log
  raw_access.log
  pr_events.log

Log every:
- Hash calculation
- CAS insert
- Raw retrieval
- PR creation

---

# 14. Backup Strategy

Host:
- Time Machine backs up BrainArchive
- GitHub backs up structured knowledge
- Optional offsite raw backup

VM:
- Periodic VM snapshot via UTM
- Or recreate from config (preferred)

Raw data should never exist only inside VM.

---

# 15. Phase Rollout Plan

Phase 0:
- Manual file ingestion
- No Gmail
- No portal automation

Phase 1:
- Media + Notes ingestion
- PR-based updates

Phase 2:
- Permit snapshot diffing
- Gmail label ingestion

Phase 3:
- Schedule auto-diff + dependency propagation

---

# 16. Security Principles

- VM is sandbox
- Repo is ledger
- Archive is immutable
- Hash is identity
- PR is gatekeeper
- Human approves structural change

---

# 17. Recovery Plan

If VM compromised:
- Shut down VM
- Rebuild from clean ISO
- Re-clone repo
- Raw data remains safe on host
- No structural knowledge lost (in GitHub)

VM should be disposable.

---

End of Document