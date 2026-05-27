# Lab 06: Listing Files and SELinux Contexts — `ls -l`, `ls -lZ`, `restorecon`, `sefcontext`

- **Series:** linux-ops-mastery — File Operations & Shell Fundamentals
- **Career arcs covered:** RHCSA EX200 (`ls -l`, `ls -lZ`, file labels), RHCE EX294 (`community.general.sefcontext`, `ansible.posix.selinux`), CKA (container SELinux types like `container_file_t`), RHCA — RH358 (custom fcontext for web roots and ports)
- **Prerequisite:** Lab 00 (Ansible control node) + Lab 05 (navigation)
- **Time Estimate:** 35–50 minutes
- **Tasks:** 5 (ADHD 3-1-1 spec — 3 RHCSA + 1 Ansible + 1 Verification capstone)
- **Practice Directory (lab-wide rotation #06):** `/etc`
- **Sandbox:** `/srv/lab06` (a non-default location so SELinux labels matter)
- **Traps rehearsed this lab:** **T01** (Forgetting `-Z` means you missed the SELinux column — invisible bug) · **T02** (`restorecon` without `-Rv` silently does nothing or only fixes one file) · **T03** (`chcon` is temporary — survives current boot but lost on relabel; only `semanage fcontext`/`sefcontext` is permanent)

> **This lab's practice directory is: `/etc`** — every task references it in at least two commands. `/srv/lab06` is the sandbox where we deliberately mislabel and fix.

---

## 🖥️ LAB HEADER BLOCK — run this FIRST

```bash
echo "🖥️  ENV:   ${ENV:-DECLARE_ME}"
echo "💿  DISK:  $(lsblk 2>/dev/null | awk '$NF=="disk"{print "/dev/"$1}' | paste -sd, -)"
echo "🌐  NIC:   $(ip -o addr show 2>/dev/null | awk '$2!="lo"{print $2}' | sort -u | paste -sd, -)"
echo "🔐  SE:    $(getenforce 2>/dev/null || echo n/a)"
echo "📦  OS:    $(cat /etc/redhat-release 2>/dev/null || grep PRETTY_NAME /etc/os-release)"
echo "🕒  TIME:  $(date -Is)"
echo "👤  USER:  $(whoami)@$(hostname)"
echo "⚠️  TRAP REMINDERS THIS LAB: T01 T02 T03"
echo "📁  PRACTICE DIR: /etc"
echo ""
echo "💡 SELinux quick-look on /etc:"
ls -dZ /etc
sestatus | head -3
```

> **STOP — confirm `getenforce` prints `Enforcing` before continuing. Permissive mode will lie to you about labels.**

---

## 🎯 Objective

Read the 10 fields of an `ls -l` listing fluently, read the 5 fields of an `ls -lZ` SELinux context fluently, and fix a mislabeled file using the **permanent** RHCSA path (`semanage fcontext` → `restorecon`) and the **permanent** RHCE path (`community.general.sefcontext` → playbook). By the end you will:

- Identify file type (`d`, `-`, `l`, `c`, `b`, `s`, `p`) from column 1 of `ls -l`
- Read DAC (mode, owner, group) AND MAC (SELinux user:role:type:level) in one glance
- Know why `chcon` is the wrong tool for a permanent fix and `sefcontext`/`semanage fcontext` is the right one
- Write an Ansible playbook that declares an fcontext rule and applies it

---

## 🛠️ Setup — run once before Task 1

```bash
sudo mkdir -p /srv/lab06
echo "lab06 content" | sudo tee /srv/lab06/page.html
sudo mkdir -p /root/rhcsa_journal/lab06
ls -lZ /srv/lab06/
```

You should see something like `unconfined_u:object_r:var_t:s0` on `/srv/lab06/page.html`. That's the **wrong** label if we wanted this to serve as a web root — Apache expects `httpd_sys_content_t`. We'll deliberately use that mismatch as the lab's running example.

---

## Task 1 — Read the 10 Fields of `ls -l` (DAC)

**Practice directory this task:** `/etc`

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab06/task1
cd /srv/lab06
date -Is | sudo tee /root/rhcsa_journal/lab06/task1/start.txt
pwd | sudo tee -a /root/rhcsa_journal/lab06/task1/start.txt
ls -l /etc | head -5 | sudo tee -a /root/rhcsa_journal/lab06/task1/start.txt
echo "exit was: $?"
```

### Purpose

Decode the 10-column `ls -l` output, distinguish files from links from directories from special files, and capture the listing to the journal.

### Main Command Block

```bash
ls -l /etc | head -10
ls -la /etc | head -10        # also show . and ..
ls -lh /etc | head -10        # human-readable sizes
ls -lF /etc | head -10        # type indicator after each name
ls -li /etc | head -10        # show inode numbers
ls -ld /etc                   # the directory itself
ls -l /etc/passwd /etc/shadow /etc/hostname

# Capture
{
  echo "=== ls -l /etc ===";      ls -l /etc | head -10
  echo "=== ls -ld /etc ===";     ls -ld /etc
  echo "=== passwd/shadow ===";   ls -l /etc/passwd /etc/shadow
  echo "=== link example ===";    ls -l /etc/grub2.cfg 2>/dev/null
} 2>&1 | sudo tee /root/rhcsa_journal/lab06/task1/transcript.txt
```

### Human-Readable Breakdown

A single `ls -l` row has 10 fields, in order:

| # | Field | Example | Meaning |
|---|---|---|---|
| 1 | Mode | `-rw-r--r--` | Type (1 char) + DAC mode (9 chars) |
| 2 | Link count | `1` | Number of hard links (≥ 2 for directories) |
| 3 | Owner | `root` | DAC owner |
| 4 | Group | `root` | DAC group |
| 5 | Size | `1234` | Bytes (use `-h` for K/M/G) |
| 6 | Month | `May` | mtime |
| 7 | Day | `27` | mtime |
| 8 | Year/Time | `15:04` or `2024` | mtime |
| 9 | Name | `passwd` | Filename |
| 10 (with `-i`) | Inode | `12345` | Filesystem inode number |

Field 1 character meanings: `-` regular file, `d` directory, `l` symlink, `c` character device, `b` block device, `s` socket, `p` named pipe.

### Reading It Left to Right

`ls -l /etc/passwd`

- `ls` — list directory
- `-l` — long format
- `/etc/passwd` — file argument

`-rw-r--r--`

- `-` — regular file
- `rw-` — owner: read + write
- `r--` — group: read
- `r--` — other: read

### The Story

A grader points at one line of `ls -l` and asks "what is this?" Your answer is the 10-field decode. If you stumble on "is it a symlink?" — that's field 1 char `l` — you've lost RHCSA points on a free question.

### Expected Output

```
$ ls -l /etc/passwd
-rw-r--r--. 1 root root 2456 May 27 14:03 /etc/passwd

$ ls -l /etc/shadow
----------. 1 root root 1234 May 27 14:03 /etc/shadow
```

Notice `/etc/shadow` is mode `000` — readable only via group/setuid. That's the RHCSA expected behavior.

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `-l` | Long format | The base of every other switch |
| `-a` | Include hidden (`.dotfiles`) | Required to see `.bashrc`, `.ssh/` etc |
| `-A` | Like `-a` but skip `.` and `..` | Cleaner for scripts |
| `-h` | Human-readable sizes | `1.2K` instead of `1234` |
| `-d` | List directory itself, not contents | `ls -ld /etc` |
| `-i` | Show inode numbers | Hard-link detection |
| `-F` | Append type indicator (`/`, `*`, `@`) | Quick type scan |
| `-t` | Sort by mtime (newest first) | Recently changed files |
| `-r` | Reverse sort | Pair with `-t` for oldest first |
| `-S` | Sort by size | Find the big file |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| DAC | Discretionary Access Control — mode, owner, group |
| Mode char 1 | File type (`-`, `d`, `l`, `c`, `b`, `s`, `p`) |
| Mode chars 2–10 | rwx triplets for owner, group, other |
| `ls -ld DIR` | Inspect the directory itself, not its contents |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T01** | Running plain `ls -l` and missing SELinux mismatches | Use `ls -lZ` whenever a service won't start, file isn't readable, etc |
| Mode-blind | Seeing `-rw-r--r--` and not knowing if it's a symlink | First char of mode is the type — burn that in |

### 🔁 Persistence Check

```bash
test -f /root/rhcsa_journal/lab06/task1/transcript.txt && echo "transcript ok"
grep -c '^[d-]' /root/rhcsa_journal/lab06/task1/transcript.txt
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab06/task1/done.txt > /dev/null <<EOF
lab=06 task=1
when=$(date -Is)
practice_dir=/etc
transcript=/root/rhcsa_journal/lab06/task1/transcript.txt
shadow_mode=$(stat -c '%a' /etc/shadow)
passwd_mode=$(stat -c '%a' /etc/passwd)
EOF
cat /root/rhcsa_journal/lab06/task1/done.txt
```

### 🧹 Cleanup

Nothing to clean — `ls` is read-only.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `ls: cannot access /etc/grub2.cfg`: `No such file or directory` | RHEL 9 + UEFI has `/boot/grub2/grub.cfg` — adjust the example |
| `ls -l` shows no `.` first column | You're on a system without SELinux installed (rare on RHEL) — `getenforce` confirms |

> **STOP — confirm transcript.txt has 10-column rows before Task 2.**

---

## Task 2 — Read the 5 Fields of `ls -lZ` (MAC — SELinux)

**Practice directory this task:** `/etc`

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab06/task2
date -Is | sudo tee /root/rhcsa_journal/lab06/task2/start.txt
getenforce | sudo tee -a /root/rhcsa_journal/lab06/task2/start.txt
ls -lZ /etc/passwd | sudo tee -a /root/rhcsa_journal/lab06/task2/start.txt
echo "exit was: $?"
```

### Purpose

Read `ls -lZ` output and decode the SELinux context label into its 4 fields (user:role:type:level). Inspect processes with `ps -eZ` to learn the matching domain types.

### Main Command Block

```bash
ls -lZ /etc/passwd /etc/shadow /etc/hostname
ls -dZ /etc /etc/httpd 2>/dev/null
ls -lZ /srv/lab06/page.html

ps -eZ | head -5
id -Z

# show the matching contexts a path *should* have per policy
matchpathcon /srv/lab06/page.html
matchpathcon /var/www/html/page.html 2>/dev/null

# Capture
{
  echo "=== ls -lZ /etc/passwd ===";     ls -lZ /etc/passwd
  echo "=== ls -dZ /etc ===";            ls -dZ /etc
  echo "=== ls -lZ sandbox ===";         ls -lZ /srv/lab06/page.html
  echo "=== matchpathcon sandbox ==="; matchpathcon /srv/lab06/page.html
  echo "=== matchpathcon /var/www ==="; matchpathcon /var/www/html/page.html 2>/dev/null
  echo "=== id -Z (your context) ==="; id -Z
} 2>&1 | sudo tee /root/rhcsa_journal/lab06/task2/transcript.txt
```

### Human-Readable Breakdown

`ls -lZ` adds the SELinux context BEFORE the filename column. A context has the shape:

```
user_u:role_r:type_t:level
```

| Field | Example | Meaning |
|---|---|---|
| user | `system_u`, `unconfined_u` | SELinux user (not the same as Linux user) |
| role | `object_r` (for files), `system_r` (for processes) | Role |
| type | `passwd_file_t`, `httpd_sys_content_t`, `var_t` | THE KEY FIELD — type enforcement happens here |
| level | `s0`, `s0-s0:c0.c1023` | MLS/MCS sensitivity (mostly `s0` on RHEL servers) |

The **type** is what matters 95% of the time. A process running with type `httpd_t` can read files of type `httpd_sys_content_t` — that's the policy match. A mislabeled web file (type `var_t` instead of `httpd_sys_content_t`) gets a Permission denied EVEN IF the DAC mode says rwx for everyone. SELinux is independent of DAC.

`matchpathcon PATH` asks the policy "what context SHOULD this path have, according to fcontext rules?" That's the diagnostic for "is my file labeled correctly?"

### Reading It Left to Right

`ls -lZ /etc/passwd`

- `ls` — list
- `-l` — long format
- `-Z` — include SELinux context column
- `/etc/passwd` — target

`system_u:object_r:passwd_file_t:s0`

- `system_u` — SELinux user (assigned by policy to system files)
- `object_r` — role for file objects (always `object_r` for files)
- `passwd_file_t` — type (only `passwd`/`shadow` utilities can write this)
- `s0` — MLS level

### The Story

A grader writes a question: "Apache returns 403 Forbidden on a page in `/srv/web/`. Fix it without changing DAC." The fastest path is: `ls -lZ /srv/web/page.html` (see `var_t`), `matchpathcon /srv/web/page.html` (see what it SHOULD be — but for `/srv/web/` policy says `var_t`, which is the wrong answer for a web file), `semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'`, `restorecon -Rv /srv/web`. Now the page works. You haven't touched chmod. That's the SELinux mindset.

### Expected Output

```
$ ls -lZ /etc/passwd
-rw-r--r--. 1 root root system_u:object_r:passwd_file_t:s0 2456 May 27 14:03 /etc/passwd

$ ls -dZ /etc
drwxr-xr-x. 90 root root system_u:object_r:etc_t:s0 8192 May 27 14:03 /etc

$ ls -lZ /srv/lab06/page.html
-rw-r--r--. 1 root root unconfined_u:object_r:var_t:s0 14 May 27 14:05 /srv/lab06/page.html

$ matchpathcon /srv/lab06/page.html
/srv/lab06/page.html  system_u:object_r:var_t:s0
```

Note the sandbox file is currently `var_t`. That's what policy expects under `/srv/lab06/` — until we add a custom rule in Task 3 and Task 4.

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `-Z` | Add SELinux context column | The whole point |
| `matchpathcon PATH` | "What context SHOULD this path have?" | Diagnostic |
| `id -Z` | Your shell's SELinux context | Tells you which domain your commands run in |
| `ps -eZ` | Process contexts | See which domain a service is in |
| `getenforce` | `Enforcing`/`Permissive`/`Disabled` | Pre-flight before any SELinux lab |
| `sestatus` | Full SELinux status (mode + policy + loaded modules) | Deeper diagnostic |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| DAC | Mode + owner + group (file permissions) |
| MAC | SELinux context (type enforcement) |
| 403 with rwx | Classic SELinux symptom — DAC is fine, MAC says no |
| `matchpathcon` | "What context SHOULD this path have?" |
| `chcon` | Set context on a file (TEMPORARY — lost on relabel) |
| `semanage fcontext` | Declare the policy rule (PERMANENT) |
| `restorecon` | Apply policy rules to a path (PERMANENT effect) |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T01** | Running `ls -l` and missing the SELinux mismatch | Use `ls -lZ` in any SELinux-suspect debug |
| **T03** | Using `chcon` and thinking the fix is permanent | `chcon` is reset on next `restorecon` or relabel — use `semanage fcontext` + `restorecon` |

### 🔁 Persistence Check

```bash
grep -c 'passwd_file_t'        /root/rhcsa_journal/lab06/task2/transcript.txt
grep -c 'matchpathcon sandbox' /root/rhcsa_journal/lab06/task2/transcript.txt
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab06/task2/done.txt > /dev/null <<EOF
lab=06 task=2
when=$(date -Is)
selinux_mode=$(getenforce)
sandbox_context=$(ls -Z /srv/lab06/page.html | awk '{print $1}')
expected_context=$(matchpathcon /srv/lab06/page.html | awk '{print $2}')
EOF
cat /root/rhcsa_journal/lab06/task2/done.txt
```

### 🧹 Cleanup

Nothing to clean.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `getenforce` prints `Permissive` | `sudo setenforce 1` (temporary) or edit `/etc/selinux/config` and reboot |
| `matchpathcon: command not found` | `sudo dnf install -y policycoreutils` |
| `ls -lZ` has no context column | SELinux is disabled — re-enable via `/etc/selinux/config` and reboot |

> **STOP — confirm `selinux_mode=Enforcing` in done.txt before Task 3.**

---

## Task 3 — RHCSA Permanent Label Fix: `semanage fcontext` + `restorecon`

**Practice directory this task:** `/etc` (`/etc/selinux/targeted/contexts/files/file_contexts.local` is where rules live)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab06/task3
date -Is | sudo tee /root/rhcsa_journal/lab06/task3/start.txt
ls -Z /srv/lab06/page.html | sudo tee -a /root/rhcsa_journal/lab06/task3/start.txt
echo "exit was: $?"
```

### Purpose

Mark `/srv/lab06` as web content with `semanage fcontext -a -t httpd_sys_content_t`, apply with `restorecon -Rv`, and verify with `ls -Z`. This is the RHCSA-grade permanent fix.

### Main Command Block

```bash
# 0) Confirm policycoreutils-python-utils is installed (semanage lives here)
rpm -q policycoreutils-python-utils || sudo dnf install -y policycoreutils-python-utils

# 1) Declare the policy rule — '/srv/lab06(/.*)?' covers the dir AND everything under it
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/lab06(/.*)?'

# 2) Show the rule was stored
sudo semanage fcontext -l | grep '/srv/lab06'

# 3) Apply the policy to existing files
sudo restorecon -Rv /srv/lab06

# 4) Verify the label
ls -lZ /srv/lab06/
matchpathcon /srv/lab06/page.html

# Capture
{
  echo "=== rule stored ===";    sudo semanage fcontext -l | grep '/srv/lab06'
  echo "=== restorecon ===";     sudo restorecon -Rv /srv/lab06
  echo "=== after ls -lZ ==="; ls -lZ /srv/lab06/
  echo "=== matchpathcon ===";   matchpathcon /srv/lab06/page.html
} 2>&1 | sudo tee /root/rhcsa_journal/lab06/task3/transcript.txt
```

### Human-Readable Breakdown

`semanage fcontext` edits the **fcontext policy database** at `/etc/selinux/targeted/contexts/files/file_contexts.local`. The rule says: "any file whose path matches `/srv/lab06(/.*)?` should have type `httpd_sys_content_t`." That rule survives reboot, package updates, and full system relabels.

`restorecon -Rv /srv/lab06` walks the path and applies the policy. Without `-R` it would only touch the leaf; without `-v` it would be silent — both are RHCSA trap-fixes.

The regex `(/.*)?` is the convention: match the directory itself OR the directory followed by any path. `(/.*)?` is roughly "optional slash-anything." Use it on every `semanage fcontext -a` of a directory tree.

### Reading It Left to Right

`semanage fcontext -a -t httpd_sys_content_t '/srv/lab06(/.*)?'`

- `semanage` — SELinux policy management tool
- `fcontext` — sub-command for file-context rules
- `-a` — **add** a rule (counterpart: `-d` delete, `-m` modify)
- `-t TYPE` — set the type to TYPE
- `'/srv/lab06(/.*)?'` — path regex, **quoted** so the shell doesn't expand `(/.*)?` as a glob

`restorecon -Rv /srv/lab06`

- `restorecon` — apply policy to existing files
- `-R` — recursive
- `-v` — verbose; prints every label change

### The Story

A grader's question: "Make `/srv/lab06` serve as a web root. SELinux must approve. Survive reboot." The two-step answer is: `semanage fcontext -a ...` (declare the rule), `restorecon -Rv ...` (apply it). Without step 1, step 2 reverts your changes the moment policy reloads. Both steps are mandatory.

### Expected Output

```
$ sudo semanage fcontext -l | grep '/srv/lab06'
/srv/lab06(/.*)?                                   all files          system_u:object_r:httpd_sys_content_t:s0

$ sudo restorecon -Rv /srv/lab06
Relabeled /srv/lab06 from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /srv/lab06/page.html from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0

$ ls -lZ /srv/lab06/
total 4
-rw-r--r--. 1 root root unconfined_u:object_r:httpd_sys_content_t:s0 14 May 27 14:05 page.html

$ matchpathcon /srv/lab06/page.html
/srv/lab06/page.html  system_u:object_r:httpd_sys_content_t:s0
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `semanage fcontext -a -t TYPE PATH` | Add a permanent fcontext rule | Survives reboot |
| `semanage fcontext -l` | List all fcontext rules | RHCSA verification |
| `semanage fcontext -d PATH` | Delete a rule | Cleanup or correction |
| `restorecon -Rv PATH` | Apply policy recursively, verbose | The "make it so" step |
| `chcon -t TYPE PATH` | Set type on a file — TEMPORARY | Useful for one-shot tests, NOT permanent |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Permanent fix | `semanage fcontext -a` + `restorecon -Rv` |
| Temporary fix | `chcon -t TYPE PATH` (lost on relabel) |
| Path regex convention | `'/path(/.*)?'` covers dir + everything under it |
| `file_contexts.local` | Where custom rules persist on disk |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T02** | `restorecon` without `-R` only fixes the leaf | Always use `-Rv` on a directory tree |
| **T03** | Using `chcon` and considering the fix done | `chcon` is temporary — graders relabel and find your context gone. Use `semanage fcontext` |
| Forgetting `()? `regex | Writing `/srv/lab06` instead of `/srv/lab06(/.*)?` — rule only applies to the directory itself, not files | Always use the `(/.*)?` suffix on a tree |

### 🔁 Persistence Check

```bash
sudo semanage fcontext -l | grep -c '/srv/lab06'
matchpathcon /srv/lab06/page.html | grep -c 'httpd_sys_content_t'
ls -Z /srv/lab06/page.html | grep -c 'httpd_sys_content_t'
```

All three must return `1`.

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab06/task3/done.txt > /dev/null <<EOF
lab=06 task=3
when=$(date -Is)
rule_stored=$(sudo semanage fcontext -l | grep -c '/srv/lab06')
matchpathcon=$(matchpathcon /srv/lab06/page.html | awk '{print $2}')
actual=$(ls -Z /srv/lab06/page.html | awk '{print $1}')
EOF
cat /root/rhcsa_journal/lab06/task3/done.txt
```

### 🧹 Cleanup

LEAVE the fcontext rule in place — Task 4 demonstrates removing it and re-applying with Ansible. We'll fully clean in Task 5.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `semanage: command not found` | `sudo dnf install -y policycoreutils-python-utils` |
| `restorecon` says no changes | Either the rule isn't stored, or files already match — check with `semanage fcontext -l` |
| `matchpathcon` still shows `var_t` | The rule didn't store — re-run `semanage fcontext -a` with the regex quoted |

> **STOP — confirm `httpd_sys_content_t` appears in `done.txt` before Task 4.**

---

## Task 4 — Ansible: Declare the fcontext Rule with `community.general.sefcontext`

**Practice directory this task:** `/etc`

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab06/task4/playbooks
date -Is | sudo tee /root/rhcsa_journal/lab06/task4/start.txt
ansible --version | head -1 | sudo tee -a /root/rhcsa_journal/lab06/task4/start.txt
ansible-galaxy collection list | grep community.general | sudo tee -a /root/rhcsa_journal/lab06/task4/start.txt
echo "exit was: $?"
```

If `community.general` isn't listed, **go back to Lab 00 Task 2** before proceeding.

### Purpose

Redo Task 3 the RHCE way: declare the fcontext rule with `community.general.sefcontext` and apply it with `ansible.builtin.command: restorecon -Rv`. Prove idempotence by running the playbook twice and confirming `changed=0` on the second run.

> **First, undo Task 3** so we have a real "before" state to drive change:

```bash
sudo semanage fcontext -d '/srv/lab06(/.*)?' 2>/dev/null || true
sudo restorecon -Rv /srv/lab06     # reverts files back toward default
ls -Z /srv/lab06/page.html         # should NOT say httpd_sys_content_t anymore
```

### Main Command Block

Write the playbook:

```bash
sudo tee /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml > /dev/null <<'EOF'
---
- name: Lab 06 Task 4 — label /srv/lab06 as web content via Ansible
  hosts: localhost
  become: true
  gather_facts: false

  vars:
    target_dir: /srv/lab06
    target_type: httpd_sys_content_t

  tasks:
    - name: Declare fcontext rule (permanent — stored in policy DB)
      community.general.sefcontext:
        target: "{{ target_dir }}(/.*)?"
        setype: "{{ target_type }}"
        state: present
      register: rule_result

    - name: Apply policy — restorecon (no Ansible module; command: is RHCE-accepted)
      ansible.builtin.command:
        cmd: "restorecon -Rv {{ target_dir }}"
      register: restore_result
      changed_when: "'Relabeled' in restore_result.stdout"

    - name: Show what changed
      ansible.builtin.debug:
        msg:
          - "rule changed: {{ rule_result.changed }}"
          - "restorecon changed: {{ restore_result.changed }}"
          - "restorecon stdout: {{ restore_result.stdout_lines }}"
EOF
```

Check-mode first:

```bash
ansible-playbook --check --diff /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab06/task4/check.log
```

Apply:

```bash
ansible-playbook /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab06/task4/apply.log
```

Idempotence proof — run AGAIN, expect `changed=0`:

```bash
ansible-playbook /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab06/task4/rerun.log
grep '^localhost' /root/rhcsa_journal/lab06/task4/rerun.log
```

### Human-Readable Breakdown

`community.general.sefcontext` is the RHCE module for the `semanage fcontext` operation. The arguments map 1:1:

- `target: '/srv/lab06(/.*)?'` ≡ the path regex from `semanage fcontext -a`
- `setype: httpd_sys_content_t` ≡ `-t httpd_sys_content_t`
- `state: present` ≡ `-a` (use `state: absent` for `-d`)

`restorecon` has **no native Ansible module**. The RHCE-accepted pattern is `ansible.builtin.command: restorecon -Rv {{ target }}` with `changed_when:` set to only flag changed when stdout contains "Relabeled" — that makes the playbook idempotent. Without `changed_when`, every run shows `changed=1` even if no relabel happened.

This is the rare case where `command:` is correct in an RHCE answer — because the operation has no module-backed alternative. We mark it explicitly in the Concept Card.

### Reading It Left to Right

```yaml
community.general.sefcontext:
  target: "{{ target_dir }}(/.*)?"
  setype: "{{ target_type }}"
  state: present
```

- `community.general.sefcontext:` — FQCN of the sefcontext module
- `target:` — path regex (same as `semanage fcontext` argument)
- `setype:` — the SELinux type (e.g. `httpd_sys_content_t`)
- `state: present` — add the rule (or `absent` to remove)

```yaml
ansible.builtin.command:
  cmd: "restorecon -Rv {{ target_dir }}"
changed_when: "'Relabeled' in restore_result.stdout"
```

- `ansible.builtin.command:` — exec a binary (NOT a shell)
- `cmd:` — the executable and args
- `changed_when:` — Jinja expression; task is marked `changed` only when it evaluates true
- `'Relabeled' in restore_result.stdout` — Python `in` check on the registered stdout

### The Story

A grader reading your playbook sees `community.general.sefcontext` (correct module), `state: present` (correct verb), `target:` with the proper regex (correct path), `restorecon` wrapped in `command:` with `changed_when:` (correct because there is no module). Then the grader runs your playbook a second time and watches `changed=0` come back. Full marks.

### Expected Output

First apply:

```
TASK [Declare fcontext rule] ***
changed: [localhost]

TASK [Apply policy — restorecon] ***
changed: [localhost]

TASK [Show what changed] ***
ok: [localhost] => {
    "msg": [
        "rule changed: True",
        "restorecon changed: True",
        "restorecon stdout: ['Relabeled /srv/lab06 from ... to ...httpd_sys_content_t...']"
    ]
}

PLAY RECAP ***
localhost : ok=3 changed=2 unreachable=0 failed=0
```

Second run (idempotence proof):

```
TASK [Declare fcontext rule] ***
ok: [localhost]                    <-- not "changed"

TASK [Apply policy — restorecon] ***
ok: [localhost]                    <-- changed_when=False because stdout has no "Relabeled"

PLAY RECAP ***
localhost : ok=3 changed=0 unreachable=0 failed=0
```

### Switches Table

| Switch / Key | Meaning | Why it matters |
|---|---|---|
| `community.general.sefcontext` | FQCN of the sefcontext module | RHCE answer for `semanage fcontext` |
| `state: present` / `absent` | Add or remove the rule | Idempotent declaration |
| `target:` | Path regex (must include `(/.*)?` for trees) | Same convention as RHCSA Task 3 |
| `setype:` | SELinux type | The whole point |
| `ansible.builtin.command:` + `changed_when:` | Acceptable wrapper for tools with no module | restorecon has no module |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| `community.general.sefcontext` | RHCE module for fcontext rules — equivalent to `semanage fcontext` |
| `ansible.builtin.command` + `changed_when` | The honest way to wrap `restorecon` — RHCE-accepted because no module exists |
| Idempotence | Second run shows `changed=0` because rule already exists AND no files needed relabeling |
| `--check --diff` | Dry-runs the playbook; useful for previewing fcontext rule additions |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **Wrapping shell commands in `command:`/`shell:` instead of using a real module** | Whole point of RHCE Ansible | Use FQCN module first; only fall back when no module exists (restorecon is one such case) |
| Missing `changed_when:` on the `command: restorecon` task | Every playbook run shows `changed=1` — not idempotent | Always pair `command:` with `changed_when:` based on stdout |
| Forgetting `state: present` | Default is `present`, but be explicit so the playbook reads correctly to graders | Always write `state: present` or `state: absent` |

### 🔁 Persistence Check

```bash
sudo semanage fcontext -l | grep -c '/srv/lab06'
ls -Z /srv/lab06/page.html | grep -c 'httpd_sys_content_t'
test -f /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml && echo "playbook ok"
grep -c 'changed=0' /root/rhcsa_journal/lab06/task4/rerun.log
```

All four must return `1`.

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab06/task4/done.txt > /dev/null <<EOF
lab=06 task=4
when=$(date -Is)
ansible_module=community.general.sefcontext
playbook=/root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml
rule_present=$(sudo semanage fcontext -l | grep -c '/srv/lab06')
file_label=$(ls -Z /srv/lab06/page.html | awk '{print $1}')
idempotent_rerun_changed_0=$(grep -c 'changed=0' /root/rhcsa_journal/lab06/task4/rerun.log)
EOF
cat /root/rhcsa_journal/lab06/task4/done.txt
```

### 🧹 Cleanup

Leave the rule in place — Task 5 verifies it and removes everything together.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `couldn't resolve module 'community.general.sefcontext'` | Lab 00 Task 2 was skipped — run `ansible-galaxy collection install community.general` |
| Second run shows `changed=1` on the restorecon task | `changed_when:` is missing or the path doesn't match — re-check the playbook |
| `policycoreutils-python-utils` missing on target | Module needs the same `python3-policycoreutils` package; install with `dnf` |

> **STOP — confirm rerun.log shows `changed=0` before Task 5.**

---

## Task 5 — RHCSA Verification Capstone: Prove the Label is Real, Permanent, and Reboot-Safe

**Practice directory this task:** `/etc/selinux/targeted/contexts/files/file_contexts.local` (the persistence file)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab06/task5
date -Is | sudo tee /root/rhcsa_journal/lab06/task5/start.txt
ls -Z /srv/lab06/page.html | sudo tee -a /root/rhcsa_journal/lab06/task5/start.txt
echo "exit was: $?"
```

### Purpose

Use **only** RHCSA inspection commands (no `ansible` CLI) to prove:

1. The file label IS `httpd_sys_content_t`
2. The rule IS stored in policy and survives reboot
3. The fix matches what `matchpathcon` predicts

### Main Command Block

Three RHCSA inspection commands plus the persistence file:

```bash
# 1) Direct inspection — current label
ls -lZ /srv/lab06/
stat -c 'mode=%a owner=%U group=%G' /srv/lab06/page.html

# 2) Policy prediction — what SHOULD it be?
matchpathcon /srv/lab06/page.html

# 3) Persistence file — is the rule actually on disk?
sudo grep '/srv/lab06' /etc/selinux/targeted/contexts/files/file_contexts.local
sudo semanage fcontext -l -C  # -C: only the customizations (custom rules)

# Diff: do the actual label and matchpathcon agree?
actual=$(ls -Z /srv/lab06/page.html | awk '{print $1}' | awk -F: '{print $3}')
expected=$(matchpathcon /srv/lab06/page.html | awk '{print $2}' | awk -F: '{print $3}')
echo "actual=$actual expected=$expected"
[ "$actual" = "$expected" ] && echo "MATCH" || echo "MISMATCH"

# Capture
{
  echo "=== ls -lZ ==="; ls -lZ /srv/lab06/
  echo "=== matchpathcon ==="; matchpathcon /srv/lab06/page.html
  echo "=== persistence file ==="; sudo grep '/srv/lab06' /etc/selinux/targeted/contexts/files/file_contexts.local
  echo "=== semanage -C ==="; sudo semanage fcontext -l -C
  echo "=== actual vs expected ==="
  actual=$(ls -Z /srv/lab06/page.html | awk '{print $1}' | awk -F: '{print $3}')
  expected=$(matchpathcon /srv/lab06/page.html | awk '{print $2}' | awk -F: '{print $3}')
  echo "actual=$actual expected=$expected"
  [ "$actual" = "$expected" ] && echo "MATCH" || echo "MISMATCH"
} 2>&1 | sudo tee /root/rhcsa_journal/lab06/task5/evidence.txt
```

### Human-Readable Breakdown

The capstone has three checks:

1. **What does `ls -lZ` see?** — current observable label
2. **What does `matchpathcon` predict?** — what policy says it should be
3. **What's in `file_contexts.local`?** — the persistence file that survives reboot

If all three agree on `httpd_sys_content_t`, the fix is real and reboot-safe. If `ls -lZ` and `matchpathcon` disagree, somebody used `chcon` (T03). If the persistence file is missing the rule, somebody only ran `restorecon` without first declaring the rule (T03's twin).

### Reading It Left to Right

`semanage fcontext -l -C`

- `semanage fcontext` — file-context management
- `-l` — list rules
- `-C` — **only** custom rules (filters out the thousands of default policy rules — RHCSA-grade noise reduction)

`awk -F: '{print $3}'`

- `awk` — text tool
- `-F:` — set field separator to `:`
- `'{print $3}'` — print field 3 (the SELinux **type**)

### The Story

A grader's audit script runs `ls -lZ`, `matchpathcon`, and `sudo grep ... file_contexts.local`. If all three print `httpd_sys_content_t`, you pass. If `ls -lZ` shows `httpd_sys_content_t` but `file_contexts.local` is empty, the grader concludes you used `chcon` (temporary) and marks you down. You match the grader's audit by doing the same audit yourself.

### Expected Output

```
=== ls -lZ ===
-rw-r--r--. 1 root root unconfined_u:object_r:httpd_sys_content_t:s0 14 May 27 14:05 page.html

=== matchpathcon ===
/srv/lab06/page.html  system_u:object_r:httpd_sys_content_t:s0

=== persistence file ===
/srv/lab06(/.*)?    system_u:object_r:httpd_sys_content_t:s0

=== semanage -C ===
SELinux fcontext                                   type               Context
/srv/lab06(/.*)?                                   all files          system_u:object_r:httpd_sys_content_t:s0

=== actual vs expected ===
actual=httpd_sys_content_t expected=httpd_sys_content_t
MATCH
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `ls -lZ` | List with SELinux context | RHCSA primary inspection |
| `matchpathcon PATH` | Policy's expected context | "What SHOULD this be?" |
| `semanage fcontext -l -C` | Custom rules only | Filters out distro defaults |
| `awk -F: '{print $3}'` | Extract the type field | Programmatic compare |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Permanent SELinux fix | rule in `file_contexts.local` + applied label on file |
| Reboot reasoning | `file_contexts.local` survives reboot; on relabel, policy reapplies your rule automatically |
| Auditor reflex | Verify with `ls -lZ` + `matchpathcon` + persistence-file grep |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **Trusting `ansible-playbook`'s "changed=1" without inspecting state** | Whole point of RHCE → RHCSA loop | Always verify with `ls -lZ`, `matchpathcon`, and `semanage -C` |
| Skipping persistence-file check | Believing label is permanent when it was set by `chcon` | Always `grep PATH file_contexts.local` |

### 🔁 Persistence Check (Reboot Reasoning)

```bash
echo "REBOOT REASONING:"                                                                            | sudo tee /root/rhcsa_journal/lab06/task5/reboot.txt
echo "1. /etc/selinux/targeted/contexts/files/file_contexts.local survives reboot (it's on disk)." | sudo tee -a /root/rhcsa_journal/lab06/task5/reboot.txt
echo "2. On boot, SELinux loads the policy + local customizations; the rule is back in effect."    | sudo tee -a /root/rhcsa_journal/lab06/task5/reboot.txt
echo "3. If a full relabel ever runs, restorecon walks the tree and re-applies httpd_sys_content_t." | sudo tee -a /root/rhcsa_journal/lab06/task5/reboot.txt
test -f /etc/selinux/targeted/contexts/files/file_contexts.local && echo "persistence file exists" | sudo tee -a /root/rhcsa_journal/lab06/task5/reboot.txt
sudo grep -c '/srv/lab06' /etc/selinux/targeted/contexts/files/file_contexts.local                 | sudo tee -a /root/rhcsa_journal/lab06/task5/reboot.txt
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab06/task5/done.txt > /dev/null <<EOF
lab=06 task=5
when=$(date -Is)
evidence=/root/rhcsa_journal/lab06/task5/evidence.txt
reboot=/root/rhcsa_journal/lab06/task5/reboot.txt
match_actual_expected=$(grep -c '^MATCH' /root/rhcsa_journal/lab06/task5/evidence.txt)
status=lab06-complete
EOF
cat /root/rhcsa_journal/lab06/task5/done.txt
```

### 🧹 Cleanup (No Regression — return system to a known state)

```bash
# Remove the rule we added
sudo semanage fcontext -d '/srv/lab06(/.*)?'

# Re-run restorecon to revert files toward default policy
sudo restorecon -Rv /srv/lab06

# Remove the sandbox directory
sudo rm -rf /srv/lab06

# Confirm no rule left and no sandbox left
sudo semanage fcontext -l -C | grep -c '/srv/lab06'   # should be 0
ls -d /srv/lab06 2>&1 | grep -q "No such" && echo "sandbox cleaned"

# The Ansible playbook stays for future reference
ls -l /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `MISMATCH` line in evidence.txt | `chcon` was used somewhere — re-run `semanage fcontext -a` and `restorecon -Rv` |
| Cleanup `semanage fcontext -d` errors with "rule not found" | Already removed; fine |
| `rm -rf /srv/lab06` errors with "Permission denied" | Use `sudo` (you should already; check `whoami`) |

> **STOP — record `status=lab06-complete` in done.txt. Lab 06 is finished.**

---

## ✅ Lab 06 Complete When

```bash
ls /root/rhcsa_journal/lab06/task{1,2,3,4,5}/done.txt
grep -l 'lab06-complete' /root/rhcsa_journal/lab06/task5/done.txt
test -f /root/rhcsa_journal/lab06/task4/playbooks/sefcontext.yml
grep -c 'MATCH' /root/rhcsa_journal/lab06/task5/evidence.txt
```

All four checks must succeed. You now own the RHCSA + RHCE SELinux fcontext loop end-to-end.
