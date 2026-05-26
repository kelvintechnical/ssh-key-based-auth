# Lab: Configure SSH and Key-Based Authentication

- **Series:** linux-ops-mastery — RHCSA Networking
- **Subjects covered:** OpenSSH server role (`sshd`), client config basics, **`ssh-keygen`** RSA key pair generation, **`ssh-copy-id`** public-key deployment, `~/.ssh` permissions (`0700`, `0600`), `authorized_keys` format, testing **`ssh -i`**, passwordless login verification, cleaning lab keys
- **Career arcs covered:** RHCSA (SSH hardening and keys appear in objectives), RHCE (Ansible `authorized_key` module), SRE (break-glass access patterns), DevOps (CI deploy keys), AI/MLOps (notebook → cluster `ssh` without passwords in scripts)
- **Prerequisite:** Two user accounts on one VM **or** two VMs on the same lab network; root/sudo access; Lab 36 networking basics so `ping` between hosts works
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 verify sshd · 2 generate RSA key · 3 deploy pubkey · 4 test and tighten permissions · 5 edge cases · 6 capstone + cleanup

---

## Objective

Replace password typing with **cryptographic proof** you control a private key file. By the end of this lab you can generate an RSA key pair suitable for legacy compatibility requirements, install the public half into `authorized_keys` on a remote account, and verify login with **`ssh -i`** while diagnosing permission mistakes from `sshd` logs.

The capstone is the RHCSA-realistic framing: *"As user `alice`, create `~/.ssh/id_rsa_lab` and `id_rsa_lab.pub`, authorize `alice` on `server1` for passwordless login, confirm `ssh -i ~/.ssh/id_rsa_lab alice@server1` works with `BatchMode yes`, then remove the lab key line and delete the key files."*

> **Lab safety note:** Never copy private keys into chat, tickets, or git repos. This lab keeps keys **local** under `~/.ssh` with `_lab` suffixes so you cannot confuse them with production identities.

---

## Concept: SSH Keys Are Asymmetric — Private Stays, Public Travels

**OpenSSH** uses public-key cryptography for authentication. You generate a **key pair**: a **private** key (secret file on your client) and a **public** key (one line you append server-side). `sshd` challenges the client to prove possession of the private key without ever transmitting it.

```
  Client (alice)                         Server (sshd)
 ┌─────────────────┐                  ┌──────────────────────┐
 │ ~/.ssh/id_rsa   │  NEVER copy      │ ~/.ssh/authorized_keys│
 │   (PRIVATE)     │  off disk        │   (PUBLIC lines)      │
 └────────┬────────┘                  └──────────┬───────────┘
          │                                       │
          │  ssh handshake proves key possession  │
          └───────────────────┬───────────────────┘
                              ▼
                     Session opens if match OK
```

> **Why this matters:** Passwords scale poorly to automation. Keys are how Ansible, `git`, `rsync`, and `scp` authenticate non-interactively — but only if permissions and ownership are correct.

---

## 📜 Why SSH Key Auth Exists — The Story

Password authentication over the network was always fragile: shoulder surfing, reusable passwords across systems, and brute-force bots on port 22. **SSH protocol version 2** (standardized work in the late 1990s, widely deployed in the 2000s) made key-based authentication practical at scale alongside symmetric session keys.

OpenSSH — the implementation on RHEL — split responsibilities cleanly: **`ssh-keygen`** manages key material, **`ssh-copy-id`** automates the `authorized_keys` append dance, **`sshd`** enforces host keys (`/etc/ssh/ssh_host_*_key`) while user keys live under each account’s home directory. The historical shift is from *"shared root password"* to *"per-user keys with sudo audit trails"*.

> **The point of the story:** Keys are not "more secure magic" — they move the secret from a memorized string to a **file permission problem**. This lab teaches both the crypto and the chmod discipline.

---

## 👪 The SSH Key Family — Who Lives There

### By file (client)

| File | Purpose |
|---|---|
| `~/.ssh/id_rsa` | Default RSA private key (legacy default name) |
| `~/.ssh/id_rsa.pub` | Matching public line |
| `~/.ssh/id_ed25519` | Modern default (not this lab’s focus) |
| `~/.ssh/config` | Per-host `IdentityFile`, `User`, `Port` |

### By file (server)

| File | Purpose |
|---|---|
| `~/.ssh/authorized_keys` | One public key per line — grants access |
| `/etc/ssh/sshd_config` | Global toggles: `PubkeyAuthentication`, `PasswordAuthentication` |
| `/var/log/secure` | RHEL auth log lines from `sshd` |

### By command

| Command | Role |
|---|---|
| `ssh-keygen -t rsa -b 4096` | Create RSA key pair |
| `ssh-copy-id -i KEY.pub USER@HOST` | Append pubkey to remote `authorized_keys` |
| `ssh -i KEY USER@HOST` | Select non-default private key |
| `ssh -o BatchMode=yes` | Fail fast if still password-prompting (great test) |

> **The point of the family tree:** If login fails, either **keys mismatch**, **permissions** are wrong, or **`sshd_config`** disallows the method.

---

## 🔬 The Anatomy of `authorized_keys` — In One Diagram

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC… rsa-key-20260526
└─┬──┘ └──────────────┬─────────────────────┘ └──┬──┘
  │                    │                            └ optional comment
  │                    └ Base64-encoded public exponent+modulus blob
  └ key type label
```

`sshd` reads the line, matches type, and ties the blob to the attempted private key proof.

> **Reading rule:** **One line per key.** Line breaks pasted from email destroy auth silently.

---

## 📚 SSH Key Reference Table

| Task | Command | Notes |
|---|---|---|
| Generate RSA key | `ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_lab -N ""` | Empty passphrase for lab automation only |
| Show fingerprint | `ssh-keygen -lf ~/.ssh/id_rsa_lab.pub` | Verify you copied right pubkey |
| Install pubkey | `ssh-copy-id -i ~/.ssh/id_rsa_lab.pub USER@HOST` | Uses password **once** |
| Test | `ssh -i ~/.ssh/id_rsa_lab USER@HOST hostname` | Remote command |
| Non-interactive test | `ssh -o BatchMode=yes -i … USER@HOST true` | Must return 0 |
| Remove key line | Edit `authorized_keys` | Delete matching line |

> **Rule one of SSH keys:** **`~/.ssh` must be mode `700`**, private keys **`600`**, `authorized_keys` **`600`**.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Objective sets include configuring SSH and securing access — keys are foundational. |
| **RHCE candidate** | Ansible `authorized_key` must match exact key strings you generate here. |
| **SRE / Platform** | Rotating host keys and user keys without lockouts mirrors this lab’s rollback. |
| **DevOps** | Deploy keys are read-only repo authentication — same pubkey mechanics. |
| **AI / MLOps** | Orchestrators `ssh` between nodes for bootstrap — keys, not passwords. |

---

## 🔧 The 6 Tasks

> Six phases: **sshd check → keygen → copy-id → verify → hardening edge → capstone**.

---

### Task 1 — Verify OpenSSH server and client, find the listening socket

**Purpose:** Confirm `sshd` is active on port 22 (or custom), and that `ssh`/`ssh-keygen` exist.

```bash
sudo -i
systemctl is-active sshd || systemctl is-active ssh
ss -lntp | grep -E ':22\s' || ss -lntp | grep sshd

rpm -q openssh-server openssh-clients
```

**Human-Readable Breakdown:** RHEL may name the unit `sshd` or `ssh` depending on version/pattern — check both. `ss` lists listening TCP with process names.

**Reading it left to right:** If `sshd` is inactive, `ssh-copy-id` cannot succeed.

**The story:** Half of “SSH is broken” tickets are **firewall** or **sshd down** — not keys.

**Expected output:**

```text
active
LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=…,fd=…))
openssh-server-8.7p1-…
openssh-clients-8.7p1-…
```

**Switches**

| Token | Meaning |
|---|---|
| `systemctl is-active` | Prints `active` or `inactive` |
| `ss -lntp` | Listening TCP, numeric ports, processes |
| `rpm -q` | Query installed package |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No listener | `systemctl enable --now sshd` |
| `ss` empty | `dnf install iproute` |
| Port not 22 | Note port; use `ssh -p` later |

---

### Task 2 — Generate an RSA lab key pair with a non-default filename

**Purpose:** Create **`~/.ssh/id_rsa_lab`** per the brief (RSA) without overwriting any existing default keys.

```bash
su - alice
mkdir -p ~/.ssh
chmod 700 ~/.ssh

ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_lab -C "alice@lab-rsa" -N ""

ls -l ~/.ssh/id_rsa_lab*
ssh-keygen -lf ~/.ssh/id_rsa_lab.pub
```

**Human-Readable Breakdown:** `alice` is a normal user for realistic permissions. `chmod 700` first — `ssh-keygen` refuses unsafe paths on modern OpenSSH. `-N ""` sets empty passphrase (**lab only**). `-C` adds comment suffix in `.pub` line.

**Reading it left to right:** RSA key generation time is noticeable at 4096 bits — that is normal.

**The story:** RSA is requested in legacy integration exams; Ed25519 is shorter and faster — learn both over your career, RSA here.

**Expected output:**

```text
Your identification has been saved in /home/alice/.ssh/id_rsa_lab
Your public key has been saved in /home/alice/.ssh/id_rsa_lab.pub
-rw-------. 1 alice alice 3243 … id_rsa_lab
-rw-r--r--. 1 alice alice  748 … id_rsa_lab.pub
4096 SHA256:… alice@lab-rsa (RSA)
```

**Switches**

| Token | Meaning |
|---|---|
| `-t rsa` | Algorithm |
| `-b 4096` | Modulus size |
| `-f PATH` | Output base path |
| `-N ""` | Passphrase (empty) |
| `-C COMMENT` | Pubkey trailing comment |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `UNPROTECTED PRIVATE KEY FILE!` | `chmod 600 ~/.ssh/id_rsa_lab` |
| `alice` missing | `useradd alice` as root and set password |

---

### Task 3 — Deploy the public key with `ssh-copy-id`

**Purpose:** Use password auth **one last time** to append the pubkey to `alice@localhost` (single-host lab) or a second VM.

**Single-host pattern (alice → alice on same box via localhost):**

```bash
ssh-copy-id -i ~/.ssh/id_rsa_lab.pub -o StrictHostKeyChecking=accept-new alice@localhost
```

**Two-VM pattern:**

```bash
# from alice@workstation:
ssh-copy-id -i ~/.ssh/id_rsa_lab.pub alice@192.0.2.10
```

**Human-Readable Breakdown:** `ssh-copy-id` wraps `ssh` to create `~/.ssh` remotely if needed and appends the `.pub` line to `authorized_keys`.

**Reading it left to right:** `-i` selects which **public** file to send. `StrictHostKeyChecking=accept-new` avoids interactive hostkey prompt in lab automation — think carefully before using in production.

**The story:** This is the “password bridge” step — afterwards you should be able to disable password auth **only** when keys are proven.

**Expected output:**

```text
Number of key(s) added: 1
Now try: ssh -i /home/alice/.ssh/id_rsa_lab alice@localhost
```

**Switches**

| Token | Meaning |
|---|---|
| `ssh-copy-id` | Helper from `openssh-clients` |
| `-i PUB` | Explicit public key path |
| `-o K=V` | Per-invocation ssh option |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied (publickey)` already | Password auth disabled — use console to seed keys |
| `ssh-copy-id: command not found` | Install `openssh-clients` |
| Still prompts password after | Wrong remote user — copy to the account you `ssh` as |

---

### Task 4 — Verify passwordless login and inspect remote files

**Purpose:** Prove **`BatchMode`** works (automation-safe) and show resulting server-side files.

```bash
ssh -o BatchMode=yes -i ~/.ssh/id_rsa_lab alice@localhost 'hostname; pwd; wc -l ~/.ssh/authorized_keys'

ssh -i ~/.ssh/id_rsa_lab alice@localhost 'grep -n "alice@lab-rsa" ~/.ssh/authorized_keys || true'
```

**Human-Readable Breakdown:** Remote commands run non-interactively. `wc -l` counts authorized keys. `grep` finds your comment.

**Reading it left to right:** `BatchMode=yes` causes immediate failure if a password would be needed — perfect CI check.

**The story:** If this passes, your key is in the right account with sane permissions.

**Expected output:**

```text
client1.lab.example.com
/home/alice
1 /home/alice/.ssh/authorized_keys
1:ssh-rsa AAAAB3… alice@lab-rsa
```

**Switches**

| Token | Meaning |
|---|---|
| `-o BatchMode=yes` | No prompts |
| `'cmd'` | Remote command string |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied (publickey)` | Remote perms: `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys` |
| Host key changed | Remove offending line from `~/.ssh/known_hosts` |

---

### Task 5 — Edge case: wrong `authorized_keys` mode and recovery via log hint

**Purpose:** Simulate world-readable `authorized_keys` (`644`), confirm **`sshd` ignores the file**, read **`/var/log/secure`**, then repair permissions **as root on disk** (pubkey login may fail until fixed).

```bash
exit   # drop to root if you were su'd as alice
sudo -i

chmod 644 /home/alice/.ssh/authorized_keys
ls -l /home/alice/.ssh/authorized_keys

sudo -u alice ssh -o BatchMode=yes -i /home/alice/.ssh/id_rsa_lab alice@localhost true \
  || echo "expected failure: sshd skips insecure authorized_keys"

grep -i "bad permissions" /var/log/secure | tail -n 3

chmod 600 /home/alice/.ssh/authorized_keys
sudo -u alice ssh -o BatchMode=yes -i /home/alice/.ssh/id_rsa_lab alice@localhost true \
  && echo "recovered"
```

**Human-Readable Breakdown:** OpenSSH requires strict ownership and modes on `authorized_keys`. World-readable files are rejected so a local non-owner cannot swap keys. Logs usually mention **bad permissions** or **insecure**.

**Reading it left to right:** Root can always `chmod` the path directly — you do not need a working `ssh` session to recover.

**The story:** This failure mode often follows `chmod -R 755 ~` advice from outdated tutorials — never relax `~/.ssh`.

**Expected output:**

```text
-rw-r--r--. 1 alice alice … /home/alice/.ssh/authorized_keys
expected failure: sshd skips insecure authorized_keys
Authentication refused: bad ownership or modes for file /home/alice/.ssh/authorized_keys
recovered
```

**Switches**

| Token | Meaning |
|---|---|
| `chmod 644` | Group/other read — triggers refusal |
| `sudo -u alice ssh …` | Test exactly as `alice` |
| `grep /var/log/secure` | RHEL `sshd` auth diagnostics |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No log line | Confirm `rsyslog` / journald path; `journalctl -u sshd -n 20` |
| Still fails after `600` | Ensure `~/.ssh` is `700` and owned by `alice` |

---

### Task 6 — Capstone: document key, prove `BatchMode`, remove lab material

**Task statement:** *"As `alice`, show the public fingerprint, prove non-interactive login to `localhost`, save `/home/alice/.ssh/id_rsa_lab.pub` into `/home/alice/lab-pubkey-backup.txt`, then remove the matching line from `authorized_keys` and delete `id_rsa_lab` + `id_rsa_lab.pub`."*

**Purpose:** End-to-end lifecycle: deploy, verify, roll back key material (not just sessions).

```bash
su - alice
ssh-keygen -lf ~/.ssh/id_rsa_lab.pub
ssh -o BatchMode=yes -i ~/.ssh/id_rsa_lab alice@localhost 'echo CAPSTONE_OK'

cp -a ~/.ssh/id_rsa_lab.pub ~/lab-pubkey-backup.txt

# Remove ONLY the lab public line (match on comment suffix):
grep -v 'alice@lab-rsa' ~/.ssh/authorized_keys > ~/.ssh/authorized_keys.new
mv ~/.ssh/authorized_keys.new ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Pubkey line is gone — lab key should no longer authenticate:
ssh -o BatchMode=yes -o PasswordAuthentication=no -i ~/.ssh/id_rsa_lab alice@localhost true \
  && echo "unexpected: key still works" || echo "expected: lab key rejected"

rm -f ~/.ssh/id_rsa_lab ~/.ssh/id_rsa_lab.pub ~/lab-pubkey-backup.txt
test ! -f ~/.ssh/id_rsa_lab && test ! -f ~/.ssh/id_rsa_lab.pub \
  && ! grep -q 'alice@lab-rsa' ~/.ssh/authorized_keys \
  && echo "VERIFY: lab key files and pubkey line removed"
```

**Human-Readable Breakdown:** Fingerprint proves which pubkey you backed up. `BatchMode` remote echo is the grader-friendly check. `grep -v` rebuilds `authorized_keys` without the lab line. **Then** prove `ssh` rejects the lab key (`PasswordAuthentication=no` avoids silent password fallback). Only after that, delete the key files and assert they are gone.

**Layer stack you built:**

```text
id_rsa_lab (private)  ─┐
id_rsa_lab.pub        ─┼─► ssh-copy-id ─► authorized_keys line
BatchMode ssh         ─┘       proves non-interactive path
```

**The story:** The exam wants **repeatable** access changes. Removing the line is as important as adding it — stale keys are a compliance finding.

**Expected output:**

```text
4096 SHA256:… alice@lab-rsa (RSA)
CAPSTONE_OK
expected: lab key rejected
VERIFY: lab key files and pubkey line removed
```

**Cleanup**

```bash
# as alice — remove localhost hostkey lab entry if desired:
ssh-keygen -R localhost 2>/dev/null || true
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `grep -v` deleted all lines | Restore from backup or `ssh-copy-id` again |
| Still logs in | Another key line remains — inspect full file |
| `rm` refused | Check `lsattr` immutability (rare) |

---

## 🔍 SSH Key Decision Guide

```
Need access to a server account?
  │
  ├── Temporary human login → password + MFA (policy permitting)
  │
  ├── Automation / Ansible / git → key pair
  │       ├── generate: ssh-keygen -t rsa -b 4096 -f PATH
  │       ├── install:  ssh-copy-id -i PATH.pub USER@HOST
  │       └── test:     ssh -o BatchMode=yes -i PATH USER@HOST true
  │
  ├── Login fails with same key
  │       ├── permissions on ~/.ssh and authorized_keys
  │       ├── wrong user (installed key on bob, ssh as alice)
  │       └── PubkeyAuthentication off in sshd_config
  │
  └── Removing access
          └── delete that pubkey line from authorized_keys (then audit)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Confirm `sshd` listening and packages installed
- [ ] 02 `ssh-keygen` RSA keypair as `alice` at `~/.ssh/id_rsa_lab`
- [ ] 03 `ssh-copy-id` pubkey to target account
- [ ] 04 `BatchMode` remote command + inspect `authorized_keys`
- [ ] 05 Break `authorized_keys` mode with `644`, read logs, fix as root
- [ ] 06 Capstone backup fingerprint, remove line, delete keys, verify failure

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `chmod 711 ~` or loose `.ssh` | Auth silently fails | `chmod 700 ~/.ssh` |
| `644` private key | Client refuses | `chmod 600` private key |
| Pasted pubkey wrapped lines | Invalid key | One line per key |
| Copied private key to server | Security incident | Remove; rotate; never again |
| Wrong `IdentityFile` | Password prompt | `ssh -i` or `~/.ssh/config` |
| SELinux user home contexts | Rare denials | `restorecon -RFv /home/alice/.ssh` |
| Disabled `PubkeyAuthentication` | Only passwords work | Enable in `sshd_config` + reload |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the permission triangle: **`700` `.ssh`**, **`600` keys + authorized_keys**.

**RHCE candidate**
- Map this lab to `ansible.posix.authorized_key` with `exclusive: no` and `state: present`.

**SRE / Platform interview**
- Discuss **host key rotation** vs **user key rotation** — orthogonal problems.

**DevOps**
- CI deploy keys: read-only, scoped repo, never shared with humans.

**AI / MLOps**
- Jupyter → HPC: `ssh-agent` forwarding vs dedicated keys — know tradeoffs.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 36 — `nmcli` | L3 must work before `ssh-copy-id` across hosts |
| Lab 37 — `/etc/hosts` | Bootstrap name for `alice@server1` |
| Lab 38 — DNS / resolver | Forward lookups for real hostnames |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
