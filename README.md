# Lab: Listing Files and SELinux Contexts — `ls -l`, `ls -Z`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** Reading `ls -l` long listings end-to-end, the SELinux context model (`user:role:type:level`), how `ls -Z` surfaces that fourth dimension, `-d` for showing a directory's own metadata, `--time=` for switching between mtime/ctime/atime, the `.` and `+` characters at the end of the mode string, and the production reflex of comparing contexts when SELinux denies access
**Career arcs covered:** RHCSA (every SELinux exam task starts with `ls -Z`), RHCE (Ansible `community.general.sefcontext` checks), SRE (incident response on httpd/sshd permission denials), DevOps (container volume context labeling), AI/MLOps (model artifact context preservation across hosts)
**Prerequisite:** Lab 05 (Directory Navigation) — you know `cd`, `pwd`, `ls -l`
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 foundation (`ls -l` reread) · 2–3 `ls -Z` and the four context fields · 4–5 comparing contexts and using `-d` correctly · 6 RHCSA exam-realistic capstone

---

## Objective

Read `ls -l` and `ls -Z` output the way an experienced engineer does — at a glance, knowing exactly which field is which and what each value means. By the end of this lab you can spot a wrong SELinux context immediately, which is the difference between solving a denial in 30 seconds and Googling for 30 minutes.

The capstone is an exam-realistic prompt: *"List every file in `/etc/ssh` with both long permissions and SELinux contexts, save it to `/root/ssh-contexts.txt`, and identify the SELinux type label used for sshd configuration files."*

> **Lab safety note:** Every command here is read-only — `ls`, `cat`, `grep`. Nothing changes a file or a context. The next lab in the SELinux track (later in the series) will show you `chcon` / `semanage fcontext` / `restorecon` — for now we are learning to **read** before we **write**.

---

## Concept: Every File Has Four Identities

A regular Linux file has three identities visible to `ls -l`:

1. **Type and permissions** — `-rwxr-xr-x`
2. **Owner / group** — `root root`
3. **Name and timestamp** — `2547 May 21 14:33 /etc/passwd`

When the filesystem is mounted on an SELinux-enforcing system (any modern RHEL/Fedora/CentOS Stream), every file also has a **fourth identity**:

4. **SELinux security context** — `system_u:object_r:passwd_file_t:s0`

That fourth identity is invisible to plain `ls -l`. You must use `ls -Z` to see it. SELinux uses that label — not the unix permissions — to decide whether a process can read the file. A file with wide-open unix permissions but a wrong SELinux type is still untouchable to a confined daemon.

```
   ┌─────────────────────────────────────────────────────┐
   │  Standard Unix layer (DAC — Discretionary)          │
   │     -rwxr-xr-x   root:root   /etc/passwd            │
   ├─────────────────────────────────────────────────────┤
   │  SELinux layer (MAC — Mandatory)                    │
   │     system_u:object_r:passwd_file_t:s0              │
   └─────────────────────────────────────────────────────┘
                       │
                       └─ Both layers must allow access.
                          Either layer can deny.
```

> **Why this matters:** When `httpd` can't read `/var/www/html/index.html` even though `chmod 644` looks correct, the answer is almost always wrong SELinux type. `ls -Z` is the very first command you type to find out.

---

## 📜 Why SELinux Contexts Exist — The Story

In **2000**, the U.S. National Security Agency released **Security-Enhanced Linux** as a research project. The problem they were solving: standard Unix permissions (DAC — Discretionary Access Control) trust the user. If `root` is compromised, every file is compromised — there is no second line of defense.

SELinux adds **MAC — Mandatory Access Control.** Every file gets a *label* (a context). Every process gets a *domain*. A central policy says which domains may touch which labels. Even `root` cannot bypass the policy — it is enforced by the kernel.

By **2003** Red Hat had integrated SELinux into RHEL. By **RHEL 5 (2007)** it was on by default. By **RHEL 7+** every confined daemon (`httpd`, `sshd`, `mysqld`, `nginx`) ran in its own SELinux domain — meaning a compromise of `httpd` could no longer touch `mysqld`'s data, even though both ran as the same Unix user.

The context labels you see in `ls -Z` are how the kernel decides "may `httpd_t` read `httpd_sys_content_t`?" Pre-defined policy ships with the OS; you customize it for your environment with `semanage fcontext` and apply changes with `restorecon`.

> **The point of the story:** SELinux is not "extra Unix permissions." It is a **separate** access-control layer that has been opt-in on every Red Hat OS for almost two decades. Knowing how to read it is half the battle. Knowing how to fix it is the other half (later labs).

---

## 👪 The Listing Family — Who Lives There

### Long-listing flags (the `-l` family)

| Flag | What it shows | When to reach for it |
|---|---|---|
| `-l` | Permissions, owner, size, mtime, name | The default detail view |
| `-a` | Include hidden files (`.dotfiles`) | When config dirs hide their `.d/` subdirs |
| `-h` | Human-readable sizes (K/M/G) | Reading sizes at a glance |
| `-i` | Inode numbers | Hard-link debugging |
| `-d` | The directory itself, not its contents | Inspecting a single directory's metadata |
| `-r` | Reverse sort | Pair with `-t` for "oldest at bottom" reversal |
| `-t` | Sort by mtime | "What changed recently?" |
| `-S` | Sort by size | Finding the biggest file |
| `-1` | One file per line | Scripting / piping |
| `--time=atime` | Use access time | Detecting "what got read recently" |
| `--time=ctime` | Use inode change time | When permissions/owner changed |

### SELinux-aware flag

| Flag | What it shows | When to reach for it |
|---|---|---|
| `-Z` | SELinux security context | Any time you suspect SELinux is involved |

### Special characters at end of mode

| Character | Meaning |
|---|---|
| (none) | No extended attributes |
| `.` | SELinux context exists (typical on modern RHEL) |
| `+` | POSIX ACL exists (Lab 47+) |

> **The point of the family tree:** `ls -l` shows the Unix layer. `ls -Z` shows the SELinux layer. Together they show you every fact the kernel uses for access decisions.

---

## 🔬 The Anatomy of `ls -Z` Output — In One Diagram

```
$ ls -Z /etc/passwd
system_u:object_r:passwd_file_t:s0  /etc/passwd
└──┬──┘ └───┬────┘ └─────┬──────┘ │
   │        │            │        └─ Path (separate column from the context)
   │        │            └─ SELinux TYPE — the field policy decisions hinge on
   │        └─ SELinux ROLE — almost always `object_r` for files
   └─ SELinux USER — almost always `system_u` for files on disk

The fourth field after the third colon is the SENSITIVITY LEVEL:
  s0    Single level (lowest, default for most files)
  s0-s1 Range (used in MLS — Multi-Level Security policy, rare outside govt)
```

Full long+context listing combines both:

```
$ ls -lZ /etc/passwd
-rw-r--r--. 1 root root system_u:object_r:passwd_file_t:s0 2547 May 21 14:33 /etc/passwd
│ │  │  │  │ │   │    │ └──────────┬───────────────────┘ │              └─ path
│ │  │  │  │ │   │    │            └─ SELinux context
│ │  │  │  │ │   │    └─ Group owner (or context if no group? no — group)
│ │  │  │  │ │   └─ User owner
│ │  │  │  │ └─ Link count
│ │  │  │  └─ '.' means SELinux context is present
│ │  │  └─ Other perms
│ │  └─ Group perms
│ └─ User perms
└─ File type (- = regular)
```

> **Reading rule:** The **type** field (third colon-separated part of the context, e.g. `passwd_file_t`) is what 99% of SELinux troubleshooting is about. Memorize the common ones: `httpd_sys_content_t` for `/var/www`, `ssh_home_t` for `~/.ssh`, `var_log_t` for `/var/log/*`, `tmp_t` for `/tmp`.

---

## 📚 Listing-with-Context Reference Table

| Task | Command | Notes |
|---|---|---|
| Long listing | `ls -l PATH` | Permissions/owner/size/time/name |
| Long listing with hidden | `ls -la PATH` | Adds `.dotfiles` |
| Long + human sizes | `ls -lh PATH` | K/M/G |
| Long listing of dir itself | `ls -ld DIR` | Shows DIR, not contents |
| SELinux context only | `ls -Z PATH` | Adds context column |
| Long + context | `ls -lZ PATH` | Most complete view |
| Long + context of dir | `ls -ldZ DIR` | Single line about the dir |
| Sort by mtime, newest last | `ls -ltr` | Best for tailing |
| Recursive listing | `ls -lR PATH` | Walks subtree |
| Show inode numbers | `ls -li PATH` | Hard-link debugging |
| Compare contexts of two paths | `ls -dZ A B` | Side-by-side |
| Show only the context | `stat -c '%C' PATH` | Scriptable single-value extraction |
| Show only the type field | `ls -Z PATH \| awk '{print $1}' \| awk -F: '{print $3}'` | Pipeline-friendly |

> **Rule one of context troubleshooting:** When permissions look right but access is denied, the answer is almost always SELinux. Always check `ls -Z` second.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | EX200 has dedicated SELinux objectives. Every fix begins with `ls -Z` — see the current value, decide if it is right. |
| **RHCE candidate** | Ansible's `community.general.sefcontext` automates label changes; you must still understand the labels themselves. |
| **SRE / Platform** | "Why can't `httpd` read this new file in `/var/www/uploads`?" → `ls -Z` shows it has type `default_t` instead of `httpd_sys_content_t`. |
| **DevOps** | Bind-mounted volumes into containers can inherit host context; `ls -Z` is the inspection tool. |
| **AI / MLOps** | Model files copied from a developer laptop to a production GPU node arrive with `user_home_t` and the inference server cannot read them — fix with restorecon. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **list → spot the context → diagnose** habit.

---

### Task 1 — Re-read `ls -l` and confirm SELinux is enforcing

**Purpose:** Refresh `ls -l` reading and confirm the system is in SELinux enforcing mode so the contexts you see are not vestigial.

```bash
mkdir -p /tmp/ctx-lab && cd /tmp/ctx-lab

ls -l /etc/passwd /etc/shadow
ls -ld /etc /var/log /home

getenforce
sestatus | head -n 5
```

**Human-Readable Breakdown:** Set up a sandbox, long-list two important system files and three system directories. Then verify SELinux is enforcing with `getenforce` and `sestatus`.

**Reading it left to right:** `ls -l FILE FILE` produces two long-listing lines. `ls -ld DIR DIR DIR` lists each directory's own metadata as a line each. `getenforce` prints one word: `Enforcing`, `Permissive`, or `Disabled`. `sestatus` prints the full status with mode and loaded policy.

**The story:** Before troubleshooting any SELinux denial you must confirm SELinux is even enabled. A "permission denied" on a machine with `getenforce` returning `Disabled` is **not** an SELinux problem.

**Expected output:**

```text
-rw-r--r--. 1 root root 2547 May 21 14:33 /etc/passwd
----------. 1 root root 1234 May 21 14:33 /etc/shadow
drwxr-xr-x. 89 root root 8192 May 26 12:48 /etc
drwxr-xr-x. 19 root root 4096 May 26 12:48 /var/log
drwxr-xr-x.  3 root root   23 May 21 14:33 /home
Enforcing
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -l FILE` | Long listing |
| `ls -ld DIR` | Long listing of the dir itself |
| `getenforce` | Print current SELinux mode |
| `sestatus` | Verbose SELinux status |
| `head -n N` | First N lines |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `getenforce` returns `Disabled` | SELinux is off entirely — re-enable via `/etc/selinux/config` and reboot |
| `getenforce` returns `Permissive` | SELinux logs denials but does not enforce — fine for diagnosis, not production |
| `Permission denied` on `/etc/shadow` | Expected — only root can read it |

---

### Task 2 — See the fourth dimension with `ls -Z`

**Purpose:** Introduce `-Z`. See exactly which extra column appears, and how it shifts the output layout.

```bash
ls -Z /etc/passwd /etc/shadow

ls -lZ /etc/passwd /etc/shadow

ls -Z /etc/ssh
ls -lZ /etc/ssh | head -n 5
```

**Human-Readable Breakdown:** Use `-Z` alone (just the path + context), then `-lZ` (full long listing + context column inserted). Then list the entire `/etc/ssh` directory contents with both views.

**Reading it left to right:** `ls -Z` produces two columns: context and path. `ls -lZ` interleaves the SELinux context **after** the group owner and **before** the size. For each file you can now see permissions / owner / context / size / time / name — six fields of metadata in one line.

**The story:** Once you have seen `-Z` you will never go back to plain `ls -l` when troubleshooting access. The extra column is the smoking gun for half the "but I gave it read permission" problems you will encounter.

**Expected output:**

```text
system_u:object_r:passwd_file_t:s0 /etc/passwd
system_u:object_r:shadow_t:s0      /etc/shadow
-rw-r--r--. 1 root root system_u:object_r:passwd_file_t:s0 2547 May 21 14:33 /etc/passwd
----------. 1 root root system_u:object_r:shadow_t:s0      1234 May 21 14:33 /etc/shadow
system_u:object_r:etc_t:s0       ssh_config
system_u:object_r:etc_t:s0       ssh_config.d
system_u:object_r:sshd_key_t:s0  ssh_host_ecdsa_key
system_u:object_r:sshd_key_t:s0  ssh_host_ed25519_key
...
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -Z` | Show SELinux context |
| `ls -lZ` | Long listing + context |
| `head -n N` | First N lines |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ls -Z` shows `?` instead of context | The filesystem is not SELinux-labeled — check `mount` |
| Context column missing | `ls --no-selinux` somewhere in shell alias — `\ls -Z` to bypass |
| Lines wrap awkwardly | Pipe to `\| less -S` for horizontal-scroll view |

---

### Task 3 — Parse the four context fields

**Purpose:** Identify each of the four colon-separated parts of an SELinux context and learn which ones matter day-to-day.

```bash
ls -Zd /var/www/html 2>/dev/null || ls -Zd /etc
ls -Zd /var/log
ls -Zd /tmp
ls -Zd /home /home/ec2-user 2>/dev/null

stat -c '%C' /etc/passwd
stat -c '%C' /var/log/messages 2>/dev/null
stat -c '%C' /tmp

ls -Zd /etc | awk '{print $1}' | awk -F: '{print "user="$1" role="$2" type="$3" level="$4}'
```

**Human-Readable Breakdown:** List the context of several well-known directories. Use `stat -c '%C'` to extract just the context string. Then split the context on `:` to label each of the four fields explicitly.

**Reading it left to right:** Each call to `ls -Zd` produces one line per path. `stat -c '%C'` prints only the context string. Piping through two `awk`s splits the line into the four colon fields and prints each with its name.

**The story:** Only the third field — the **type** — matters for 99% of troubleshooting. `system_u` and `object_r` are nearly constant for on-disk files. The level (`s0`) is constant on `targeted` policy (the RHEL default). The whole game is recognizing which type belongs to which directory.

**Expected output:**

```text
system_u:object_r:httpd_sys_content_t:s0 /var/www/html
system_u:object_r:var_log_t:s0           /var/log
system_u:object_r:tmp_t:s0               /tmp
system_u:object_r:home_root_t:s0         /home
unconfined_u:object_r:user_home_dir_t:s0 /home/ec2-user
system_u:object_r:passwd_file_t:s0
system_u:object_r:var_log_t:s0
system_u:object_r:tmp_t:s0
user=system_u role=object_r type=etc_t level=s0
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -Zd DIR` | Context of the dir itself |
| `stat -c '%C'` | Extract context as one string |
| `awk -F:` | Split on `:` |
| `\|\| ls -Zd /etc` | Fallback path if first does not exist |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `/var/www/html` does not exist | `httpd` is not installed; pick a different path |
| Field count differs | Some labels have more colons in the level (`s0:c0.c1023`) — that's fine |
| Type value looks "default_t" | The path has no rule in policy — fix with `semanage fcontext` later |

---

### Task 4 — Compare contexts and spot the wrong one

**Purpose:** Compare contexts side-by-side. Real-world SELinux denials almost always look like "the file I just created has the wrong type for the daemon that wants to read it."

```bash
cd /tmp/ctx-lab

# Two files: one in the system location, one in /tmp
ls -Z /etc/ssh/sshd_config
ls -Z /tmp

# Pretend we are deploying a custom config
sudo install -m 644 /etc/ssh/sshd_config /tmp/sshd_config_copy
ls -Z /etc/ssh/sshd_config /tmp/sshd_config_copy

# Same file content, different SELinux types — that's the bug
echo "type from /etc/ssh:"
stat -c '%C' /etc/ssh/sshd_config | awk -F: '{print $3}'
echo "type from /tmp:"
stat -c '%C' /tmp/sshd_config_copy | awk -F: '{print $3}'

sudo rm -f /tmp/sshd_config_copy
```

**Human-Readable Breakdown:** Compare two paths. Inspect a "real" sshd config (under `/etc/ssh`) and `/tmp`. Then copy the config into `/tmp` and observe that the copy inherits `tmp_t` instead of `sshd_config_t` — that mismatch is exactly the bug pattern that fails SELinux denial cases.

**Reading it left to right:** `install -m 644 SRC DST` copies SRC to DST with given mode (and does NOT preserve SELinux context — `cp -a` does). After the copy, `ls -Z` shows that the copy carries the **destination directory's** type, not the source's. That is why moving files between directories so often breaks SELinux access.

**The story:** Almost every SELinux denial in production goes like this: somebody copies a config file from one location to another, the new copy inherits the destination's default type, the daemon that's confined to read only its expected type denies access, and the error message is the unhelpful "Permission denied" even though `chmod` looks correct. The fix is `restorecon -v PATH` (later lab).

**Expected output:**

```text
system_u:object_r:etc_t:s0 /etc/ssh/sshd_config
system_u:object_r:tmp_t:s0 /tmp
system_u:object_r:etc_t:s0  /etc/ssh/sshd_config
unconfined_u:object_r:user_tmp_t:s0 /tmp/sshd_config_copy
type from /etc/ssh:
etc_t
type from /tmp:
user_tmp_t
```

**Switches**

| Token | Meaning |
|---|---|
| `install -m MODE SRC DST` | Copy with explicit mode (does not preserve context) |
| `cp -a SRC DST` | Archive copy (preserves context) |
| `stat -c '%C'` | Context only |
| `awk -F: '{print $3}'` | Third colon-separated field (the type) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Both files have the same context | The destination filesystem rule matches the source — no bug to see here |
| `install` failed | You typed `intall` — re-run |
| The copy has context `default_t` | The location has no policy match; use `restorecon` after creating |

---

### Task 5 — Use `-d` correctly and avoid the "I listed contents by accident" trap

**Purpose:** Practice the `-d` flag to list a single directory's metadata without spelunking its contents. This matters because directories carry their own context, and that context (not their contents') determines what default context new files inherit.

```bash
cd /tmp/ctx-lab

# Without -d — lists contents (potentially noisy)
ls -lZ /etc | head -n 5

# With -d — just the directory's own line
ls -ldZ /etc
ls -ldZ /var/www /var/www/html 2>/dev/null
ls -ldZ /tmp /var/tmp
ls -ldZ /home /root

# Recursive long+context within a small directory
ls -lZR /etc/ssh | head -n 20
```

**Human-Readable Breakdown:** Show the difference between `ls -lZ DIR` (contents) and `ls -ldZ DIR` (the directory itself). Compare directory contexts across `/tmp`, `/var/tmp`, `/home`, `/root`. Finish with a small recursive context dump.

**Reading it left to right:** Without `-d`, `ls -l` of a directory lists the files **inside** it. With `-d`, you get exactly one line about the directory. `-R` recurses. For SELinux work you almost always want `-ldZ` so you see only the targeted directory's own context.

**The story:** Directory context determines the **default** SELinux type of files created inside it. If `/var/www/html` is `httpd_sys_content_t`, a new file created there inherits `httpd_sys_content_t`. If you copy the file (without `-a`), you break the inheritance. Always inspect the **directory** context to predict what a new file will get.

**Expected output:**

```text
drwxr-xr-x. 4 root root system_u:object_r:etc_t:s0 80 May 21 14:33 ./adjtime
... (contents)
drwxr-xr-x. 89 root root system_u:object_r:etc_t:s0 8192 May 26 12:48 /etc
drwxr-xr-x. 4 root root system_u:object_r:httpd_sys_content_t:s0 35 ... /var/www
drwxr-xr-x. 2 root root system_u:object_r:httpd_sys_content_t:s0 6  ... /var/www/html
drwxrwxrwt. 7 root root system_u:object_r:tmp_t:s0 4096 ... /tmp
drwxrwxrwt. 3 root root system_u:object_r:tmp_t:s0 4096 ... /var/tmp
drwxr-xr-x. 3 root root system_u:object_r:home_root_t:s0 23 ... /home
dr-xr-x---. 4 root root system_u:object_r:admin_home_t:s0 215 ... /root
/etc/ssh:
total 188
-rw-r--r--. 1 root root system_u:object_r:etc_t:s0     ... ssh_config
...
```

**Switches**

| Token | Meaning |
|---|---|
| `-d` | Show the dir itself, not its contents |
| `-l` | Long listing |
| `-Z` | SELinux context |
| `-R` | Recursive |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-d` listed every file anyway | You omitted it after `-l` — write `-ldZ` together |
| Recursive listing too big | Use a narrower path or pipe through `head` |
| Sticky-bit `t` confusing on `/tmp` | The `t` at end of `drwxrwxrwt` is the sticky bit — Lab 43 |

---

### Task 6 — Capstone: RHCSA-realistic context audit

**Task statement:** *"List every file in `/etc/ssh` with both long permissions and SELinux contexts. Save the listing to `/root/ssh-contexts.txt`. Then identify which SELinux type label is used by sshd configuration files and which type is used by the host key files, and write those two type names to `/root/ssh-types.txt`."*

**Purpose:** Execute a real exam-style listing-plus-context audit end-to-end and produce the verification artifacts.

```bash
sudo -i

ls -laZ /etc/ssh > /root/ssh-contexts.txt
wc -l /root/ssh-contexts.txt
head -n 5 /root/ssh-contexts.txt

# Extract distinct types
awk '{print $4}' /root/ssh-contexts.txt \
  | awk -F: '{print $3}' \
  | sort -u \
  | tee /root/ssh-types.txt

# Explicit identification
echo "type for sshd_config:    $(stat -c '%C' /etc/ssh/sshd_config | awk -F: '{print $3}')"
echo "type for host key files: $(stat -c '%C' /etc/ssh/ssh_host_ed25519_key | awk -F: '{print $3}')"

test -s /root/ssh-contexts.txt && echo "VERIFY: contexts file exists and is non-empty"
test -s /root/ssh-types.txt && echo "VERIFY: types file exists and is non-empty"
```

**Human-Readable Breakdown:** Save the full long+hidden+context listing of `/etc/ssh` to `/root/ssh-contexts.txt`. Extract the unique set of SELinux types across all files in the directory using `awk` (4th column is the context, 3rd colon field of that is the type). Save those into `/root/ssh-types.txt`. Print the specific types for `sshd_config` and a host key.

**Layer stack you built:**

```text
/root/ssh-contexts.txt
   └── ls -laZ /etc/ssh                    <- full per-file context listing

/root/ssh-types.txt
   └── unique SELinux types seen in /etc/ssh
       ├── etc_t                           ← config files
       ├── sshd_key_t                      ← host key private files
       └── ssh_keysign_exec_t (sometimes)  ← helper binary symlink
```

**The story:** This is the **canonical 5-minute exam answer.** Memorize the spine: `ls -laZ /target/path > /root/file → awk extract type → sort -u → verify`. SELinux exam tasks almost always reduce to "save this listing" or "fix this type" — and both start exactly like this.

**Expected verification output:**

```text
18 /root/ssh-contexts.txt
total 188
drwxr-xr-x.  4 root root system_u:object_r:etc_t:s0           175 May 21 14:33 .
drwxr-xr-x. 89 root root system_u:object_r:etc_t:s0          8192 May 26 12:48 ..
-rw-r--r--.  1 root root system_u:object_r:etc_t:s0          1872 May 21 14:33 ssh_config
-rw-------.  1 root root system_u:object_r:etc_t:s0          4434 May 21 14:33 sshd_config
...
etc_t
sshd_key_t
type for sshd_config:    etc_t
type for host key files: sshd_key_t
VERIFY: contexts file exists and is non-empty
VERIFY: types file exists and is non-empty
```

**Cleanup**

```bash
rm -rf /tmp/ctx-lab
rm -f /root/ssh-contexts.txt /root/ssh-types.txt
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `awk '{print $4}'` printed paths instead of contexts | Field numbering differs because `-Z` shifts columns — adjust to the field you actually see (often `$5`); `stat -c '%C'` is safer |
| Types file has duplicates | You forgot `sort -u` |
| `Permission denied` reading `/etc/ssh/ssh_host_*_key` | You are not root — `sudo -i` first |
| File empty | Redirection went to wrong path — use absolute `/root/...` |

---

## 🔍 Listing & Context Decision Guide

```
Need to inspect a file/dir's metadata?
  │
  ├── "Just permissions, owner, size, time"
  │       └── ✅ ls -l PATH
  │
  ├── "Add hidden files"
  │       └── ✅ ls -la PATH
  │
  ├── "I want SELinux context too"
  │       └── ✅ ls -lZ PATH
  │
  ├── "What's the context of THIS DIRECTORY (not its contents)?"
  │       └── ✅ ls -ldZ DIR
  │
  ├── "Find what changed recently"
  │       └── ✅ ls -ltr        (mtime, newest at bottom)
  │       └── ✅ ls -ltcr       (ctime — permission/owner changes)
  │
  ├── "Scriptable single-value context"
  │       └── ✅ stat -c '%C' PATH
  │
  ├── "Compare contexts of two files"
  │       └── ✅ ls -dZ A B
  │
  └── "Recursively dump contexts of a subtree"
          └── ✅ ls -lZR PATH
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/ctx-lab`, re-read `ls -l`, confirm SELinux is enforcing with `getenforce` / `sestatus`
- [ ] 02 Use `ls -Z` and `ls -lZ` to see the SELinux context column
- [ ] 03 Identify the four context fields (user / role / type / level) using `stat -c '%C'` and `awk`
- [ ] 04 Compare contexts: copy a config into `/tmp` and observe the type changes from `etc_t` to `user_tmp_t`
- [ ] 05 Use `-d` to list a directory's own metadata; recurse with `-R`
- [ ] 06 Execute the RHCSA capstone — `ls -laZ /etc/ssh > /root/ssh-contexts.txt` and extract distinct types to `/root/ssh-types.txt`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Plain `ls -l` shown when troubleshooting SELinux denial | Cannot see context | Use `ls -lZ` |
| Forgot `-d` when inspecting a directory | Got contents listing instead of dir metadata | Add `-d` (so `-ldZ`) |
| Copied a file with `cp` instead of `cp -a` | Lost SELinux context | Use `cp --preserve=context` or `cp -a`, or run `restorecon` after |
| Read the 3rd column instead of 4th | Picked owner not context | `-Z` shifts columns; use `stat -c '%C'` to be safe |
| Saw `?` in context column | Filesystem not SELinux-labeled | Check `mount` and policy |
| Assumed SELinux off means no context | Labels still exist on disk | `getenforce` shows mode, not whether labels are present |
| Edited `/etc/selinux/config` then expected immediate effect | Mode changes require reboot for `disabled` ↔ `enforcing` | Use `setenforce 0/1` for runtime mode |
| Used `setenforce 0` permanently | Setting reverts on reboot | Edit `/etc/selinux/config` for persistence |
| Looked at user field (`system_u`) for clues | Almost always identical | Always check the **type** field |
| Compared types across different policies | Labels may not match | Stay on `targeted` policy unless required otherwise |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Always default to `ls -lZ` when an exam task mentions any service that "runs as a confined daemon" (httpd, sshd, vsftpd, named, nfsd). The context column tells you whether labels are right.

**RHCE candidate**
- Ansible's `community.general.sefcontext` module is your `semanage fcontext` analogue. You still need to know the right types to set.

**SRE / Platform interview**
- "Walk me through diagnosing an httpd 403 on a newly-deployed file." → "`ls -Z` the file, then `ls -ldZ /var/www/html`, then run `restorecon -v` if the file's type differs from the directory's default."

**DevOps**
- Container bind mounts on RHEL hosts need `:Z` or `:z` on the volume to relabel. Without `ls -Z` you can't verify it worked.

**AI / MLOps**
- Inference services running under SELinux confined domains cannot read model files copied from elsewhere unless contexts are corrected. `ls -lZ` is the first command in that triage.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory Navigation | Prerequisite — you must `cd` to the path before you `ls -Z` it |
| Lab 09 — Hard and Soft Links | `ls -i` (inodes) and `ls -l` symlink markers |
| Lab 14 — File Searching with `find` | `find` supports `-context` predicate for SELinux searches |
| Lab — SELinux Modes (`getenforce`, `setenforce`, `sestatus`) *(later)* | The next step in the SELinux track |
| Lab — `chcon` / `semanage fcontext` / `restorecon` *(later)* | What to do when `ls -Z` reveals the wrong context |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
