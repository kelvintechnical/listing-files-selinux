# Lab 06: Listing Files and SELinux Contexts — `ls -l`, `ls -Z`

**Series:** File Operations & Shell Fundamentals · **Lab 6 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (Tasks 4, 8, 12, 16), RHCE EX294 (Ansible `file`/`copy` module), CKA (container file labels), RHCA building blocks (RH342, RH362 IdM, RH358 services)  
**Prerequisite:** Lab 05 (Navigation)  
**Time Estimate:** 40–50 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–13 practical · 14–18 advanced · 19–20 exam-realistic

---

## 🎯 Objective

Read every column of `ls -l` fluently — every character, every dot. Then master the SELinux variant `ls -Z` so you can audit and fix security contexts. Permissions are the #1 cause of "it worked on my box but not the exam" failures, and SELinux context errors are the #2 cause.

---

## 🧠 Concept: Two Stacked Security Layers

Linux enforces access in two layers — and **both** must allow an operation:

| Layer | What it controls | Shown by |
|---|---|---|
| **DAC** — Discretionary Access Control | Owner/group/other read-write-execute bits | `ls -l` |
| **MAC** — Mandatory Access Control (SELinux) | Type-enforcement labels (e.g., `httpd_sys_content_t`) | `ls -Z` |

A file can be `rwxrwxrwx` (everyone allowed by DAC) and **still** be denied by SELinux. Likewise, a perfectly labeled file with `000` permissions is unreadable.

```
Request → DAC check (permissions) → MAC check (SELinux) → Granted/Denied
```

> **Why this matters on RHCSA Task 16:** Move the document root to `/web` and Apache returns `403 Forbidden` until you set the SELinux type to `httpd_sys_content_t`. DAC alone is not enough.

### The anatomy of `ls -l` in one diagram

```
-rw-r--r--.  1  root  root   2876  Sep  3 10:42  /etc/passwd
│└┬┘└┬┘└┬┘│  │   │     │      │         │            │
│ │  │  │ │  │   │     │      │         │            └─ Name
│ │  │  │ │  │   │     │      │         └─ Last modified
│ │  │  │ │  │   │     │      └─ Size (bytes)
│ │  │  │ │  │   │     └─ Group owner
│ │  │  │ │  │   └─ User owner
│ │  │  │ │  └─ Hard link count
│ │  │  │ └─ Special: . = SELinux, + = ACL, none = neither
│ │  │  └─ Other  permissions (r--)
│ │  └─ Group  permissions (r--)
│ └─ Owner  permissions (rw-)
└─ File type (-, d, l, c, b, s, p)
```

### The anatomy of `ls -Z`

```
system_u : object_r : httpd_sys_content_t : s0
└──┬───┘ └───┬────┘ └─────────┬─────────┘ └┬┘
  user      role            type         level
```

> **The type field is what enforces policy.** Apache reads only files of type `httpd_sys_content_t`. MariaDB touches only `mysqld_db_t`. Get the type wrong and the service fails.

---

## 📚 Command Reference

| Command | Purpose | Critical flags |
|---|---|---|
| `ls` | List | `-l`, `-a`, `-h`, `-d`, `-Z`, `-i`, `-R` |
| `stat` | Detailed file metadata, format-friendly | `-c FORMAT`, `-L` (follow symlinks) |
| `getfacl` | View POSIX ACLs (the `+` indicator) | `-p` (preserve `/`), `-R` (recursive) |
| `lsattr` | View ext/xfs file attributes (immutable etc.) | `-a` |
| `ps -eZ` | See SELinux context of running processes | none |
| `id -Z` | See your own SELinux context | none |
| `matchpathcon` | Show the **expected** SELinux context for a path | `-V` (verify) |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Tasks 4 (SELinux), 8 (permissions), 12 (ACL), 16 (web/SELinux) |
| **RHCE EX294** | Ansible `file`/`copy`/`template` modules set `mode`, `owner`, `setype` — these are the columns you're reading here |
| **CKA** | Container files under `/var/lib/containerd/`, `/var/lib/kubelet/pods` have specific contexts |
| **RHCA — RH342** | "Why is this service failing?" → 50% of the time, wrong `ls -lZ` |
| **RHCA — RH362 IdM** | Identity Management adds extra MLS/MCS labels — same syntax |
| **RHCA — RH358 services** | Every networked service has a tight SELinux policy |

---

## 🔧 The 20 Tasks

---

### Task 1 — Set up the lab workspace

**Purpose:** Get a clean working directory so the rest of the tasks have predictable output.

```bash
mkdir -p ~/listing-lab
cd ~/listing-lab
pwd
```

**Expected output:**

```
/home/ec2-user/listing-lab
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir` | **M**ake **dir**ectory |
| `-p` | **P**arents — create missing parents; don't error if it exists |
| `cd` | Change directory |
| `pwd` | Confirm location |

**Output decoded**

| Line | What it tells you |
|---|---|
| `/home/ec2-user/listing-lab` | Working directory was created and you moved into it |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied` on `mkdir` | Pick a path inside your home: `mkdir -p ~/listing-lab` |

---

### Task 2 — Create varied file types for inspection

**Purpose:** A long listing is only useful if there's something to list. Create four different types: regular file, log file, directory, symbolic link.

```bash
touch file_a.txt file_b.log
mkdir subdir
ln -s file_a.txt link_a
ls
```

**Expected output:**

```
file_a.txt  file_b.log  link_a  subdir
```

**Switches**

| Token | Meaning |
|---|---|
| `touch` | Create empty files (Lab 07) |
| `mkdir` | Create directory |
| `ln -s` | Create symbolic link (Lab 09) |
| `ls` | List (no flags) |

**Output decoded**

| Token | Meaning |
|---|---|
| `file_a.txt` | Regular file |
| `file_b.log` | Regular file (the `.log` doesn't change anything; extensions are convention only) |
| `link_a` | Symlink to `file_a.txt` — you can't see that from plain `ls` |
| `subdir` | Subdirectory |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Some entries are red | A dangling symlink — verify target exists |

---

### Task 3 — Run `ls -l` and identify file types

**Purpose:** The first character of each row is the file **type** flag. Learn it once, recognize it forever.

```bash
ls -l
```

**Expected output:**

```
total 0
-rw-r--r--. 1 ec2-user ec2-user  0 Sep 12 10:00 file_a.txt
-rw-r--r--. 1 ec2-user ec2-user  0 Sep 12 10:00 file_b.log
lrwxrwxrwx. 1 ec2-user ec2-user 10 Sep 12 10:00 link_a -> file_a.txt
drwxr-xr-x. 2 ec2-user ec2-user  6 Sep 12 10:00 subdir
```

**Switches**

| Flag | Meaning |
|---|---|
| `-l` | Long listing — one row per entry, full metadata |

**File-type characters (first column)**

| Char | Type | Where you see it |
|---|---|---|
| `-` | Regular file | Most files |
| `d` | Directory | `subdir` |
| `l` | Symbolic link | `link_a` |
| `c` | Character device | `/dev/tty0` |
| `b` | Block device | `/dev/sda` |
| `s` | Socket | `/run/*.sock` |
| `p` | Named pipe (FIFO) | Created with `mkfifo` |

**Output decoded**

| Token | Meaning |
|---|---|
| `total 0` | Sum of disk blocks (1 KiB units) used by listed files — 0 because files are empty |
| `-rw-r--r--.` | Regular file, owner read/write, group read, other read, SELinux context present |
| `lrwxrwxrwx.` | Symlink (`l`) — note: symlink permissions are ignored; the target's permissions are what matter |
| `link_a -> file_a.txt` | The arrow shows the symlink's target |
| `drwxr-xr-x.` | Directory, owner full, group r-x, other r-x |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| You see no `.` at end of permissions | SELinux disabled or filesystem doesn't support it — proceed but expect Lab 16 to behave differently |

---

### Task 4 — Decode the permission bits

**Purpose:** The 9 permission bits split into three trios: owner, group, other. Learn to read them at a glance.

```bash
chmod 750 file_a.txt
chmod 644 file_b.log
chmod 700 subdir
ls -l file_a.txt file_b.log subdir
```

**Expected output:**

```
-rwxr-x---. 1 ec2-user ec2-user 0 Sep 12 10:00 file_a.txt
-rw-r--r--. 1 ec2-user ec2-user 0 Sep 12 10:00 file_b.log
drwx------. 2 ec2-user ec2-user 6 Sep 12 10:00 subdir
```

**Switches**

| Token | Meaning |
|---|---|
| `chmod NNN` | Set mode using octal (each digit = one trio) |
| `750` | Owner = 7 (rwx), Group = 5 (r-x), Other = 0 (---) |
| `644` | Owner = 6 (rw-), Group = 4 (r--), Other = 4 (r--) |
| `700` | Owner = 7 (rwx), Group = 0, Other = 0 |

**Octal → symbolic translation table**

| Octal | Binary | Symbolic | Meaning |
|---|---|---|---|
| 0 | 000 | --- | nothing |
| 1 | 001 | --x | execute only |
| 2 | 010 | -w- | write only |
| 3 | 011 | -wx | write + execute |
| 4 | 100 | r-- | read only |
| 5 | 101 | r-x | read + execute |
| 6 | 110 | rw- | read + write |
| 7 | 111 | rwx | full |

**On directories the meanings change:**

| Bit | File | Directory |
|---|---|---|
| `r` | Read contents | List entries |
| `w` | Modify contents | Create/rename/delete entries |
| `x` | Execute | Enter (`cd`) into it |

**Output decoded**

| Row | What it tells you |
|---|---|
| `-rwxr-x---.` | Octal 750 — owner full, group r-x, other none |
| `-rw-r--r--.` | Octal 644 — common for "everyone can read" config files |
| `drwx------.` | Octal 700 on a directory — only the owner can enter or modify |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `chmod` says `Operation not permitted` | You don't own the file; need `sudo` |

---

### Task 5 — Decode the trailing `.` (SELinux indicator)

**Purpose:** The character after the 9 permission bits tells you about extra security attributes.

```bash
ls -l file_a.txt
```

**Expected output:**

```
-rwxr-x---. 1 ec2-user ec2-user 0 Sep 12 10:00 file_a.txt
```

**The 11th character demystified**

| Char | Meaning |
|---|---|
| `.` (dot) | File has an **SELinux** context |
| `+` (plus) | File has a **POSIX ACL** (extra ACL entries beyond owner/group/other) |
| (space/nothing) | Neither — usually on filesystems mounted without SELinux/ACL support |

**Output decoded**

| Token | Meaning |
|---|---|
| `.` at position 11 | SELinux is enforcing and this file has a label |

**Why a sysadmin needs this:** On RHEL family in enforcing mode, you'll see `.` everywhere. On a freshly mounted NFS share, you may see nothing.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Some files have `.`, others don't | Different filesystems mounted differently — check `mount` output |

---

### Task 6 — Trigger and decode the `+` (ACL indicator)

**Purpose:** When a file has POSIX ACL entries beyond the standard owner/group/other, `ls -l` puts a `+` after the permission bits.

```bash
setfacl -m u:nobody:rx file_a.txt
ls -l file_a.txt
getfacl file_a.txt
```

**Expected output:**

```
-rwxr-x---+ 1 ec2-user ec2-user 0 Sep 12 10:00 file_a.txt
# file: file_a.txt
# owner: ec2-user
# group: ec2-user
user::rwx
user:nobody:r-x
group::r-x
mask::r-x
other::---
```

**Switches**

| Token | Meaning |
|---|---|
| `setfacl` | Set **ACL** entries |
| `-m` | **M**odify the ACL (add/change entries) |
| `u:nobody:rx` | Grant user `nobody` read+execute |
| `getfacl` | Read the ACL |

**Output decoded**

| Line | What it tells you |
|---|---|
| `-rwxr-x---+` | Trailing `+` = additional ACLs exist |
| `# file: file_a.txt` | Header from `getfacl` |
| `user::rwx` | Owner ACL entry (the empty `:` slot is for "the owner") |
| `user:nobody:r-x` | **Named user** ACL we just added |
| `mask::r-x` | The effective rights mask — caps group/named-ACL entries |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `setfacl: nobody: No such user` | Use a real user — try `setfacl -m u:$USER:rx` |
| No `+` shows up | ACL identical to existing mode — `getfacl` may show no extra entries |

---

### Task 7 — Show human-readable sizes with `-h`

**Purpose:** Big files in bytes are unreadable. `-h` adds `K`, `M`, `G`.

```bash
dd if=/dev/zero of=file_b.log bs=1M count=2 2>/dev/null
ls -lh file_b.log
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user 2.0M Sep 12 10:05 file_b.log
```

**Switches**

| Flag | Meaning |
|---|---|
| `-l` | Long listing |
| `-h` | Human-readable sizes (1K = 1024) |
| `--si` | Alternative: SI units (1K = 1000) |
| `dd` | Disk dump — used here just to create a 2 MB file |
| `bs=1M` | Block size = 1 MiB |
| `count=2` | Write 2 blocks |

**Output decoded**

| Token | Meaning |
|---|---|
| `2.0M` | ≈ 2,097,152 bytes (2 × 1024²) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Want decimal MB (1000²)? | `ls -l --si` |

---

### Task 8 — Show only the directory with `-d`

**Purpose:** Verify a directory's own permissions without descending into it.

```bash
ls -ld subdir
ls -ld /etc
```

**Expected output:**

```
drwx------. 2 ec2-user ec2-user   6 Sep 12 10:00 subdir
drwxr-xr-x. 145 root root 8192 Sep 10 09:00 /etc
```

**Switches**

| Flag | Meaning |
|---|---|
| `-d` | **D**irectory — show the directory entry itself, not contents |
| `-l` | Long listing |

**Output decoded**

| Token | Meaning |
|---|---|
| `drwx------.` | Mode 700 — only owner can enter |
| `2` | Hard link count: `.` and `..` and any subdirs each contribute |
| `145` (on `/etc`) | `/etc` has many subdirs; each contributes to its link count |
| `8192` | Directory metadata size (not files inside) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| You forgot `-d` and see all contents | `ls -ld /etc` (single `d`, before the `l`) |

---

### Task 9 — Show hidden files in a long listing with `-la`

**Purpose:** Combine flags. Configuration files (`.bashrc`, `.ssh/`) need `-a` to be visible.

```bash
ls -la
```

**Expected output:**

```
total 2056
drwxr-xr-x.  3 ec2-user ec2-user   80 Sep 12 10:05 .
drwx------. 12 ec2-user ec2-user 4096 Sep 12 10:05 ..
-rwxr-x---+  1 ec2-user ec2-user    0 Sep 12 10:00 file_a.txt
-rw-r--r--.  1 ec2-user ec2-user 2.0M Sep 12 10:05 file_b.log
lrwxrwxrwx.  1 ec2-user ec2-user   10 Sep 12 10:00 link_a -> file_a.txt
drwx------.  2 ec2-user ec2-user    6 Sep 12 10:00 subdir
```

**Switches**

| Flag | Meaning |
|---|---|
| `-l` | Long listing |
| `-a` | All — include `.`, `..`, and dotfiles |

**Output decoded**

| Row | Meaning |
|---|---|
| `.` | The current directory (`~/listing-lab`) — same inode as the dir you're in |
| `..` | The parent (`~`) — mode 700 (private home) |
| `file_a.txt` | Note the `+` from Task 6 — still there |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Use `-A` in scripts | Skips `.` and `..` so loops don't recurse on themselves |

---

### Task 10 — See SELinux context with `ls -Z`

**Purpose:** This is the security label that policy uses. Without it, you cannot diagnose 50% of "service won't start" issues.

```bash
ls -Z file_a.txt
```

**Expected output:**

```
unconfined_u:object_r:user_home_t:s0 file_a.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-Z` | Show SELinux context |

**Output decoded — `user:role:type:level`**

| Field | Value here | Meaning |
|---|---|---|
| `unconfined_u` | SELinux **user** | "unconfined" = no extra restrictions |
| `object_r` | SELinux **role** | Almost always `object_r` for files |
| `user_home_t` | SELinux **type** | The label that policy actually checks |
| `s0` | MLS/MCS **level** | Default sensitivity |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ls: --Z: option requires...` | You spelled it lowercase; `-Z` is the only correct form on GNU ls |
| Empty output | SELinux disabled — check `sestatus` |

---

### Task 11 — Combine DAC + MAC with `ls -lZ`

**Purpose:** One row that shows you everything: type, permissions, owner, group, SELinux context, size, mtime, name.

```bash
ls -lZ file_a.txt
```

**Expected output:**

```
-rwxr-x---+ 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0 0 Sep 12 10:00 file_a.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-l` | Long listing |
| `-Z` | Add SELinux context column |

**Output decoded — full row in 9 fields**

| Field | Value | Meaning |
|---|---|---|
| 1 | `-rwxr-x---` | Type + perms |
| 2 | `+` | ACL present |
| 3 | `1` | Hard link count |
| 4 | `ec2-user` | Owner |
| 5 | `ec2-user` | Group |
| 6 | `unconfined_u:object_r:user_home_t:s0` | SELinux context |
| 7 | `0` | Size |
| 8 | `Sep 12 10:00` | mtime |
| 9 | `file_a.txt` | Name |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Row gets too wide | Use `ls -lZ \| less -S` to scroll horizontally |

---

### Task 12 — Compare SELinux types across system directories

**Purpose:** Different directories have different default types. Knowing them helps you debug "wrong directory" deployments.

```bash
ls -Zd /var/www/html /etc/ssh /home/ec2-user /tmp /var/log
```

**Expected output:**

```
system_u:object_r:httpd_sys_content_t:s0 /var/www/html
system_u:object_r:etc_t:s0               /etc/ssh
unconfined_u:object_r:user_home_dir_t:s0 /home/ec2-user
system_u:object_r:tmp_t:s0               /tmp
system_u:object_r:var_log_t:s0           /var/log
```

**Output decoded**

| Path | Type | Used by |
|---|---|---|
| `/var/www/html` | `httpd_sys_content_t` | Apache **must** see this type or returns 403 |
| `/etc/ssh` | `etc_t` | sshd reads its configs |
| `/home/user` | `user_home_dir_t` | User session processes |
| `/tmp` | `tmp_t` | World-writable temp |
| `/var/log` | `var_log_t` | rsyslog, journald, audit daemon |

**Why a sysadmin needs this on RHCSA Task 16:** Move content into `/var/www/html` and Apache returns 403 until you label it `httpd_sys_content_t`. Solved in Lab 16 (semanage + restorecon).

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Types look unfamiliar (e.g. `default_t`) | Path isn't in the policy file_contexts DB — define one with `semanage fcontext` |

---

### Task 13 — Inspect a process context with `ps -eZ` and `id -Z`

**Purpose:** SELinux is bidirectional. Files have types; processes have domains. Policy says which domains can act on which types.

```bash
ps -eZ | grep sshd | head -1
id -Z
```

**Expected output:**

```
system_u:system_r:sshd_t:s0-s0:c0.c1023 1234 ? 00:00:00 sshd
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

**Switches**

| Token | Meaning |
|---|---|
| `ps -e` | Show **e**very process |
| `-Z` | Show SELinux context column |
| `id -Z` | Show **your** current process's SELinux context |

**Output decoded**

| Field | Meaning |
|---|---|
| `sshd_t` | Domain of the sshd process |
| `unconfined_t` | Your shell's domain — "unconfined" = no special restrictions |
| `s0-s0:c0.c1023` | MLS range — for systems using categorical security |

**SELinux's golden rule:** A process **type/domain** is allowed to act on a file **type** only if policy explicitly allows it.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `id -Z` shows `failed to get current context` | SELinux is disabled — `sestatus` to confirm |

---

### Task 14 — Show the **expected** context with `matchpathcon`

**Purpose:** Policy stores expectations: "files under `/var/www` should be `httpd_sys_content_t`." `matchpathcon` reports them.

```bash
matchpathcon /var/www/html/index.html
matchpathcon /home/ec2-user/.bashrc
```

**Expected output:**

```
/var/www/html/index.html	system_u:object_r:httpd_sys_content_t:s0
/home/ec2-user/.bashrc	unconfined_u:object_r:user_home_t:s0
```

**Switches**

| Token | Meaning |
|---|---|
| `matchpathcon` | Print the **policy-defined** context for a path (no flag needed) |
| `-V` | **V**erify — compare expected vs actual, report mismatches |

**Output decoded**

| Path | Expected type |
|---|---|
| `/var/www/html/index.html` | `httpd_sys_content_t` |
| `/home/ec2-user/.bashrc` | `user_home_t` |

**Why a sysadmin needs this on RHCSA Task 16:** Run `matchpathcon -V` against suspect files; mismatches mean a missed `restorecon` after a `cp` or `mv`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `matchpathcon: command not found` | `sudo dnf install policycoreutils` |

---

### Task 15 — Use `stat` for the comprehensive metadata view

**Purpose:** `ls -l` shows one timestamp (mtime). `stat` shows access, modification, change, and birth times — plus device, inode, blocks, and context.

```bash
stat file_a.txt
```

**Expected output:**

```
  File: file_a.txt
  Size: 0           Blocks: 0          IO Block: 4096   regular empty file
Device: 0,30        Inode: 1048577     Links: 1
Access: (0750/-rwxr-x---)  Uid: ( 1000/ec2-user)   Gid: ( 1000/ec2-user)
Context: unconfined_u:object_r:user_home_t:s0
Access: 2026-09-12 10:00:00.000000000 -0400
Modify: 2026-09-12 10:00:00.000000000 -0400
Change: 2026-09-12 10:05:00.000000000 -0400
 Birth: 2026-09-12 10:00:00.000000000 -0400
```

**Output decoded**

| Field | Meaning |
|---|---|
| `Size: 0` | File contains 0 bytes |
| `Blocks: 0` | Disk blocks allocated |
| `IO Block: 4096` | Filesystem block size for I/O |
| `Device: 0,30` | Major,minor numbers of the device |
| `Inode: 1048577` | Unique inode |
| `Links: 1` | Hard link count |
| `Access (0750/-rwxr-x---)` | Mode in octal and symbolic — both shown |
| `Uid/Gid` | Numeric and named owner/group |
| `Context:` | SELinux context |
| `Access:` | atime — last time read |
| `Modify:` | mtime — last time content changed |
| `Change:` | ctime — last time inode metadata changed |
| `Birth:` | btime — when the file was created |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Birth: -` | Older filesystem doesn't track it — ext4/XFS modern installs do |

---

### Task 16 — Format `stat` output for scripts with `-c`

**Purpose:** Scripts need parseable, single-line output. `stat -c` lets you pick exactly which fields.

```bash
stat -c '%U %G %a %A %s %n' file_a.txt file_b.log
stat -c 'name=%n mode=%a context=%C' file_a.txt
```

**Expected output:**

```
ec2-user ec2-user 750 -rwxr-x--- 0 file_a.txt
ec2-user ec2-user 644 -rw-r--r-- 2097152 file_b.log
name=file_a.txt mode=750 context=unconfined_u:object_r:user_home_t:s0
```

**`stat -c` format specifiers cheat-sheet**

| Token | Meaning |
|---|---|
| `%n` | File name |
| `%s` | Size in bytes |
| `%a` | Mode in octal |
| `%A` | Mode in symbolic form |
| `%U` / `%u` | Owner user name / UID |
| `%G` / `%g` | Owner group name / GID |
| `%i` | Inode |
| `%h` | Number of hard links |
| `%C` | SELinux context |
| `%x` / `%y` / `%z` / `%w` | atime / mtime / ctime / btime |

**Output decoded**

| Field | Meaning |
|---|---|
| `750` | Octal mode |
| `-rwxr-x---` | Symbolic mode |
| `2097152` | Size of the 2 MB file in bytes |

**Why a sysadmin needs this:** One-line audit on the RHCSA exam: `stat -c '%U %G %a %C %n' /var/www/html` answers "owner/group/mode/context" in a single readable row.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Output has tabs you don't want | Quote the format string carefully; use spaces explicitly |

---

### Task 17 — See inode numbers with `-i`

**Purpose:** Inode numbers are the **disk-level identity** of a file. Different inode = different file. Same inode = same file (hard link).

```bash
ls -li file_a.txt link_a
```

**Expected output:**

```
1048577 -rwxr-x---+ 1 ec2-user ec2-user 0 Sep 12 10:00 file_a.txt
1048578 lrwxrwxrwx. 1 ec2-user ec2-user 10 Sep 12 10:00 link_a -> file_a.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-l` | Long listing |
| `-i` | **I**node — show inode number in the first column |

**Output decoded**

| Token | Meaning |
|---|---|
| `1048577` | `file_a.txt`'s inode |
| `1048578` | `link_a`'s inode — **different** from the target's, because a symlink is its own file |
| `link_a -> file_a.txt` | Arrow shows the link's target string |

**Why a sysadmin needs this:** Hard links share an inode (Lab 09); symlinks do not. `-i` is your first diagnostic.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Inode numbers vastly different | Normal — they're allocated sequentially per filesystem |

---

### Task 18 — Audit a tree recursively with `ls -RZ`

**Purpose:** Verify every file under a directory has the expected SELinux context.

```bash
sudo ls -RZ /etc/ssh
```

**Expected output (truncated):**

```
/etc/ssh:
system_u:object_r:etc_t:s0         moduli
system_u:object_r:etc_t:s0         ssh_config
system_u:object_r:etc_t:s0         ssh_config.d
system_u:object_r:etc_t:s0         sshd_config
system_u:object_r:etc_t:s0         sshd_config.d

/etc/ssh/ssh_config.d:
system_u:object_r:etc_t:s0  05-redhat.conf

/etc/ssh/sshd_config.d:
system_u:object_r:etc_t:s0  50-redhat.conf
```

**Switches**

| Flag | Meaning |
|---|---|
| `-R` | Recursive |
| `-Z` | Show SELinux context |

**Output decoded**

| Element | Meaning |
|---|---|
| `/etc/ssh:` | Header for the parent dir |
| Each context | Identical `etc_t` type — all match policy |
| Blank line | Visual separator between directory blocks |

**Why a sysadmin needs this:** After a `restorecon -R`, verify with `ls -RZ` that every file inherited the expected type.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| One file has a different context | That's the bug — `restorecon -v <file>` to fix |

---

### Task 19 — Find files by attribute with `find` + `ls`

**Purpose:** `ls` lists what you see; `find` finds files matching criteria, then `ls -l` shows them in detail.

```bash
find . -type f -perm /u+w -exec ls -lZ {} \;
find . -type l -exec ls -lZ {} \;
```

**Expected output:**

```
-rwxr-x---+ 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0 0 Sep 12 10:00 ./file_a.txt
-rw-r--r--. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0 2097152 Sep 12 10:05 ./file_b.log
lrwxrwxrwx. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0 10 Sep 12 10:00 ./link_a -> file_a.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `find .` | Search starting in current dir |
| `-type f` | Regular files only |
| `-type l` | Symbolic links only |
| `-perm /u+w` | Files where owner write bit is set (any of the listed bits) |
| `-exec ... {} \;` | Run a command on each match; `{}` substitutes the file path |

**Output decoded**

| Line | Meaning |
|---|---|
| First two rows | Files where owner has write permission |
| Third row | The symlink — note the type `l` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find: missing argument to -exec` | You forgot the literal `\;` terminator |

---

### Task 20 — Exam-style audit: verify a service path

**Task statement (RHCSA-style):** *"Confirm `/var/www/html` is owned by `root:root`, has mode `0755`, and the SELinux type is `httpd_sys_content_t`. Then verify with one report-ready line."*

```bash
sudo ls -ldZ /var/www/html
stat -c '%U %G %a %C %n' /var/www/html
matchpathcon /var/www/html
```

**Expected output:**

```
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_content_t:s0 6 Aug  9 12:00 /var/www/html
root root 755 system_u:object_r:httpd_sys_content_t:s0 /var/www/html
/var/www/html	system_u:object_r:httpd_sys_content_t:s0
```

**Step-by-step audit**

| Check | Source | Pass criterion |
|---|---|---|
| Owner = `root` | `ls -l` col 3 or `stat %U` | `root` |
| Group = `root` | `ls -l` col 4 or `stat %G` | `root` |
| Mode = `0755` | `stat %a` | `755` |
| Context type = `httpd_sys_content_t` | `ls -Z` or `stat %C` | matches |
| Policy alignment | `matchpathcon` | actual matches expected |

**Output decoded**

| Row | Pass/fail |
|---|---|
| `drwxr-xr-x.` | Permissions = `0755` — PASS |
| `root root` | Ownership — PASS |
| `httpd_sys_content_t` | Context — PASS |
| `matchpathcon` matches `ls -Z` | Policy alignment — PASS |

**Cleanup**

```bash
cd ~
rm -rf ~/listing-lab
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Mode shows `755`, task says `0755` | They mean the same — the leading `0` is the octal prefix |
| Context differs from `matchpathcon` | Run `sudo restorecon -Rv /var/www/html` |

---

## 🔍 Listing Decision Guide

```
Just names?                    → ls
Names + hidden?                → ls -a
Full metadata?                 → ls -l
Sizes I can read?              → ls -lh
The directory itself?          → ls -ld
SELinux contexts?              → ls -Z   (or ls -lZ)
Sorted by time (newest last)?  → ls -ltrh
Inode numbers?                 → ls -li
One per line for scripts?      → ls -1A
Everything, everywhere?        → ls -RlaZ   (use sparingly)
Audit-ready one-liner?         → stat -c '%U %G %a %C %n' <file>
Policy-defined context?        → matchpathcon <path>
Diff actual vs expected?       → matchpathcon -V <path>
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/listing-lab`
- [ ] 02 Create file, log, dir, symlink samples
- [ ] 03 Identify file types from `ls -l` column 1
- [ ] 04 Decode permission bits (`chmod 750/644/700`)
- [ ] 05 Explain the trailing `.` (SELinux)
- [ ] 06 Trigger and read the `+` (ACL)
- [ ] 07 Use `-h` for human-readable sizes
- [ ] 08 Use `-d` to show the directory itself
- [ ] 09 Combine `-la` to see hidden files in long format
- [ ] 10 `ls -Z` shows `user:role:type:level`
- [ ] 11 `ls -lZ` for combined DAC + MAC
- [ ] 12 Compare contexts across `/var/www`, `/etc/ssh`, `/home`, `/tmp`, `/var/log`
- [ ] 13 Inspect process contexts with `ps -eZ` and `id -Z`
- [ ] 14 Query expected context with `matchpathcon`
- [ ] 15 `stat` shows atime/mtime/ctime/btime
- [ ] 16 `stat -c` builds audit-ready one-liners
- [ ] 17 `ls -li` reveals inode numbers
- [ ] 18 `ls -RZ` audits a tree
- [ ] 19 `find -exec ls -lZ` combines criteria + listing
- [ ] 20 Exam scenario: audit `/var/www/html`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Misreading `rwxr-xr--` | Granting wrong access in a script | Read in **3-bit groups**: owner, group, other |
| Ignoring the trailing `.` | Surprised by SELinux denial later | `.` = SELinux is **on** — always run `ls -Z` |
| `ls -l` on a directory expecting metadata | Floods with contents | Add `-d`: `ls -ld /etc` |
| Forgetting `-Z` after restoring contexts | Can't confirm fix worked | Always re-verify with `ls -lZ` after `restorecon` |
| `cp /tmp/foo /etc/foo` | New file inherits wrong type → service breaks | Use `cp -a`, `cp -Z`, or run `restorecon` (Lab 08) |
| Sizes in bytes for big files | Hard to compare 8192 vs 8389120 | Always add `-h` |
| Trusting `ls -l` alone for security audits | Miss SELinux/ACL | Use `ls -lZ` and `getfacl` |
| Reading the `+` as ACL but having no rules | Pre-existing ACL from a `cp -a` | Inspect with `getfacl` to confirm |

---

## 📌 Exam Strategy

**RHCSA EX200**
- After **every** permission task, verify with one line: `stat -c '%U %G %a %C %n' <file>`.
- Trailing `.` is a free reminder SELinux is enforcing — don't ignore it.
- When a graded service won't start, compare contexts: `ls -Z <bad>` vs `ls -Z <known-good>`.

**RHCE EX294 (Ansible)**
- The `file` module's `mode`, `owner`, `group`, `seuser`, `serole`, `setype`, `selevel` map **exactly** to fields shown by `ls -lZ`.

**CKA**
- Static-pod manifests must be `root:root`, mode `0644` — verify with `ls -l /etc/kubernetes/manifests/`.
- Containerd state can have unexpected SELinux types after a restore — `ls -Z` first.

**RHCA**
- RH342: `ls -lZ` + `matchpathcon -V` is the SELinux troubleshooting first move.
- RH362 IdM: identity-aware paths use the same listing model.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory navigation | You must `cd` somewhere before listing |
| Lab 07 — `touch` | Create files to list |
| Lab 09 — Hard/soft links | Uses `ls -li` to compare inodes |
| Lab 12 — ACLs | Deep dive on the `+` indicator and `getfacl`/`setfacl` |
| Lab 16 — Apache + SELinux | Hands-on `chcon` / `semanage fcontext` / `restorecon` |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
