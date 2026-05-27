# Lab 06: Listing Files and SELinux Contexts — `ls -l`, `ls -Z`

- **Series:** linux-ops-mastery — File Operations & Shell Fundamentals
- **Permanent standard:** every lab is exactly **5 tasks**. If a topic needs more, split it into another 5-task lab.
- **Practice Directory (rotation #06):** `/etc`
- **Sandbox:** `/tmp/lsZ-lab`
- **Journal:** `/root/rhcsa_journal/lab06/`
- **Traps rehearsed:** **T01** setting permissive instead of fixing context, **T02** `semanage fcontext` without `restorecon -Rv`, **T03** `setsebool` without `-P`.

> **This lab's practice directory is: `/etc`** — every task references `/etc` in at least two commands.

---

## Objective

Read every important column of `ls -l`, then layer SELinux context awareness on top with `ls -Z`, `id -Z`, `ps -eZ`, `matchpathcon`, `semanage fcontext -l`, and `restorecon -nRv`. You will practice without changing production files, then journal progress so the lab can be resumed after reboot.

---

## Lab Header Block

Run this first and confirm or correct the environment values before Task 1.

```bash
sudo -i
echo "ENV:   ${ENV:-DECLARE_BAREMETAL_OR_AMI}"
echo "DISK:  $(lsblk | awk '$6=="disk"{print "/dev/"$1}' | paste -sd, -)"
echo "NIC:   $(ip -o addr show | awk '$2!="lo"{print $2}' | sort -u | tr -d ':' | paste -sd, -)"
echo "SE:    $(getenforce 2>/dev/null || echo unavailable)"
echo "OS:    $(cat /etc/redhat-release 2>/dev/null || grep PRETTY_NAME /etc/os-release)"
echo "TIME:  $(date -Is)"
echo "USER:  $(whoami)@$(hostname)"
echo "TRAPS: T01 T02 T03"
echo "PRACTICE DIR: /etc"
```

> **STOP — paste the header output before setup.** If `getenforce` says `Disabled`, complete the DAC portions now and redo SELinux tasks on Rocky/RHEL with SELinux enforcing.

---

## Concept: DAC and MAC Must Both Allow Access

| Layer | Tool | What it answers |
|---|---|---|
| DAC — Discretionary Access Control | `ls -l`, `chmod`, `chown` | Do Unix owner/group/other bits allow access? |
| MAC — Mandatory Access Control | `ls -Z`, `semanage`, `restorecon`, `setsebool` | Does SELinux policy allow this process type to touch this file type? |

A file can be `rwxrwxrwx` and still be blocked by SELinux. The field that usually matters most in an SELinux context is the **type**, such as `etc_t`, `passwd_file_t`, `httpd_sys_content_t`, or `ssh_home_t`.

---

## Lab-Wide Setup

```bash
sudo -i
mkdir -p /tmp/lsZ-lab /root/rhcsa_journal/lab06
cd /tmp/lsZ-lab

cat > /tmp/lsZ-lab/THIS_DIRECTORY.txt <<'EOF'
/etc — System-wide configuration files

/etc holds text configuration for services, boot behavior, authentication,
networking, storage mounts, SELinux, firewalld, chrony, SSH, and many other
subsystems. Binaries do not live here; configuration and identity do.

Why it exists: packages need a predictable place for administrator-editable
configuration. Backing up /etc is close to backing up the system's identity.

What lives inside it: passwd, group, shadow, fstab, ssh/, selinux/, systemd/,
chrony.conf, firewalld/, hostname, hosts, and service config directories.

Why RHCSA cares: the exam constantly asks you to inspect or modify files in
/etc, and mistakes here can break login, boot, networking, and service access.
EOF

cat /tmp/lsZ-lab/THIS_DIRECTORY.txt
echo "setup done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

> **STOP — paste setup output before Task 1.**

---

# The 5 Tasks

---

## Task 1 — Read `ls -l` Columns on `/etc`

**Practice directory this task:** `/etc`

`/etc` contains high-value configuration files whose ownership and permissions must be read correctly before any change.

### 🔁 Warm-Up — Commands from Previous Labs

```bash
cd /tmp/lsZ-lab
date -Is > task01-warmup.log
echo "user=$(whoami) host=$(hostname) kernel=$(uname -r)" >> task01-warmup.log
ls /etc 2>/dev/null | wc -l >> task01-warmup.log
ls /etc/passwd /etc/group > task01-files.txt 2>&1
cat task01-warmup.log task01-files.txt | tee task01-warmup.copy
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Decode the long-listing columns for core `/etc` files: file type, mode, link count, owner, group, size, timestamp, and name.

### Main Command Block

```bash
cd /tmp/lsZ-lab
ls -l /etc/passwd /etc/group /etc/hostname > task01-long.txt 2>&1
cat task01-long.txt
awk '{print $1, $3, $4, $5, $9}' task01-long.txt | tee task01-columns.txt
echo "lines: $(wc -l < task01-long.txt)"
test -s task01-long.txt && echo "long listing saved"
```

### Human-Readable Breakdown

- `ls -l` prints one metadata row per target.
- `/etc/passwd`, `/etc/group`, and `/etc/hostname` are safe read-only targets.
- `awk` extracts selected columns: mode, owner, group, size, and path.
- `wc -l < file` counts rows without printing a filename.
- `test -s` confirms the evidence file is non-empty.

### Reading It Left to Right

`ls -l /etc/passwd /etc/group /etc/hostname > task01-long.txt 2>&1`

1. `ls` lists metadata.
2. `-l` requests long format.
3. The three `/etc` paths are targets.
4. `>` redirects stdout to a file.
5. `2>&1` merges stderr into that same file.

### The Story

Before SELinux even enters the picture, you must read DAC permissions fluently. The first ten characters of `ls -l` tell you file type and owner/group/other access. On RHCSA, this is your first evidence line.

### Expected Output

```text
-rw-r--r--. 1 root root ... /etc/passwd
-rw-r--r--. 1 root root ... /etc/group
-rw-r--r--. 1 root root ... /etc/hostname
-rw-r--r--. root root ... /etc/passwd
lines: 3
long listing saved
```

### Switches Table

| Token | Meaning |
|---|---|
| `ls -l` | Long listing with metadata columns |
| `>` | Redirect stdout and truncate target file |
| `2>&1` | Send stderr to stdout's current destination |
| `awk '{print ...}'` | Print selected fields |
| `tee` | Display and save output |
| `wc -l < file` | Pure line count |
| `test -s` | True if file exists and is non-empty |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | DAC mode | Owner/group/other bits from `ls -l` |
|  | Owner/group | Columns 3 and 4 in long listing |
|  | `.` after mode | SELinux context exists on many RHEL files |
|  | Evidence capture | Save output before cleanup |
|  | `$?` | Last command status |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| T01 | You see a permission denial later and set SELinux permissive instead of reading file evidence | Always capture `ls -l` and `ls -Z` first |

### 🔁 Persistence Check

| What was configured | Verification command | Why it matters |
|---|---|---|
| Evidence file | `test -s /tmp/lsZ-lab/task01-long.txt` | confirms current output exists |
| `/etc` metadata | `ls -l /etc/passwd` | survives reboot because `/etc` is persistent |
| Journal root | `test -d /root/rhcsa_journal/lab06` | confirms reboot-safe location |

```bash
test -s /tmp/lsZ-lab/task01-long.txt && echo "evidence=OK"
ls -l /etc/passwd
test -d /root/rhcsa_journal/lab06 && echo "journal=OK"
```

### 📓 Journal Write — Before Cleanup

```bash
LAB=lab06
TASK=task1
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"
cat > "$JDIR/done.txt" <<EOF
LAB:    lab06
TASK:   task1
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname)
STATUS: COMPLETE
EOF
cat > "$JDIR/notes.txt" <<EOF
TOPIC:    Read ls -l columns on /etc files
COMMANDS: ls -l, awk, tee, wc -l, test -s
TRAPS:    T01
MISSED:   none
NEXT:     task2 — file types and permission characters
EOF
ls -la "$JDIR"
echo "exit was: $?"
```

### 🧹 Cleanup

```bash
rm -f /tmp/lsZ-lab/task01-warmup.log /tmp/lsZ-lab/task01-files.txt /tmp/lsZ-lab/task01-warmup.copy /tmp/lsZ-lab/task01-long.txt /tmp/lsZ-lab/task01-columns.txt
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `/etc/hostname` missing | Use `/etc/hosts` instead |
| `awk` prints blank path | Some filenames contain spaces; use `stat` for machine parsing later |
| Journal write fails | Run `sudo -i` |

> **STOP — paste output before Task 2.**

---

## Task 2 — Identify File Types and Permission Characters

**Practice directory this task:** `/etc`

`/etc` has regular files, directories, and symlinks. The sandbox adds controlled examples you can inspect safely.

### 🔁 Warm-Up — Commands from Previous Labs

```bash
cd /tmp/lsZ-lab
find /etc -maxdepth 1 -type f 2>/dev/null | head -n 5 | tee task02-etc-files.txt
find /etc -maxdepth 1 -type d 2>/dev/null | head -n 5 | tee task02-etc-dirs.txt
grep -c '^/' task02-etc-files.txt
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Read the first character of `ls -l` (`-`, `d`, `l`) and the permission triplets (`rwx`, `rw-`, `r-x`, `---`).

### Main Command Block

```bash
cd /tmp/lsZ-lab
touch regular.txt
mkdir -p sampledir
ln -s regular.txt symlink.txt
chmod 644 regular.txt
chmod 755 sampledir
ls -l regular.txt sampledir symlink.txt /etc/passwd /etc/ssh | tee task02-types.txt
awk '{print $1, $9}' task02-types.txt | tee task02-first-column.txt
```

### Human-Readable Breakdown

- `touch` creates a regular file.
- `mkdir -p` creates a directory and does not fail if it already exists.
- `ln -s` creates a symlink.
- `chmod 644` and `chmod 755` create predictable permissions.
- `ls -l` shows file type and permission bits.
- `awk` extracts the mode string and name for focused reading.

### Reading It Left to Right

`ls -l regular.txt sampledir symlink.txt /etc/passwd /etc/ssh | tee task02-types.txt`

1. `ls -l` requests metadata rows.
2. Each path is a target.
3. `| tee` displays the rows and saves them.

### The Story

The first character in `ls -l` answers “what kind of thing is this?” before you read anything else: `-` regular file, `d` directory, `l` symlink. The next nine characters are the access model you will change with `chmod` in later labs.

### Expected Output

```text
-rw-r--r--. ... regular.txt
drwxr-xr-x. ... sampledir
lrwxrwxrwx. ... symlink.txt -> regular.txt
-rw-r--r--. ... /etc/passwd
drwxr-xr-x. ... /etc/ssh
```

### Switches Table

| Token | Meaning |
|---|---|
| `touch` | Create empty file or update timestamp |
| `mkdir -p` | Create directory safely |
| `ln -s` | Create symbolic link |
| `chmod 644` | Owner rw, group r, other r |
| `chmod 755` | Owner rwx, group rx, other rx |
| `find -type f` | Match regular files |
| `find -type d` | Match directories |
| `head -n 5` | First five lines |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `-` file type | Regular file |
|  | `d` file type | Directory |
|  | `l` file type | Symlink |
|  | `rwx` triplets | Owner/group/other permissions |
|  | Symlink mode | Usually ignored; target permissions matter |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| T01 | You blame SELinux for a DAC permission problem | Read `ls -l` first; SELinux is second layer, not first |

### 🔁 Persistence Check

| What was configured | Verification command | Why it matters |
|---|---|---|
| Sandbox file modes | `ls -l /tmp/lsZ-lab/regular.txt /tmp/lsZ-lab/sampledir` | confirms chmod results |
| Symlink target | `ls -l /tmp/lsZ-lab/symlink.txt` | confirms link arrow points to regular.txt |
| Reboot survival | `ls /tmp/lsZ-lab` after reboot | expected to fail; sandbox is temporary |

```bash
ls -l /tmp/lsZ-lab/regular.txt /tmp/lsZ-lab/sampledir /tmp/lsZ-lab/symlink.txt
```

### 📓 Journal Write — Before Cleanup

```bash
LAB=lab06
TASK=task2
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"
cat > "$JDIR/done.txt" <<EOF
LAB:    lab06
TASK:   task2
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname)
STATUS: COMPLETE
EOF
cat > "$JDIR/notes.txt" <<EOF
TOPIC:    File types and permission characters
COMMANDS: touch, mkdir -p, ln -s, chmod 644, chmod 755, ls -l
TRAPS:    T01
MISSED:   none
NEXT:     task3 — SELinux contexts with ls -Z
EOF
ls -la "$JDIR"
echo "exit was: $?"
```

### 🧹 Cleanup

```bash
rm -f /tmp/lsZ-lab/task02-etc-files.txt /tmp/lsZ-lab/task02-etc-dirs.txt /tmp/lsZ-lab/task02-types.txt /tmp/lsZ-lab/task02-first-column.txt /tmp/lsZ-lab/regular.txt /tmp/lsZ-lab/symlink.txt
rm -rf /tmp/lsZ-lab/sampledir
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `ln: failed to create symbolic link` | Remove old link with `rm -f symlink.txt` and retry |
| `chmod` denied | You are not root or path is wrong; run inside `/tmp/lsZ-lab` |
| Symlink shows `lrwxrwxrwx` | Normal; symlink permissions are not the target's permissions |

> **STOP — paste output before Task 3.**

---

## Task 3 — Read SELinux Contexts with `ls -Z`, `id -Z`, and `ps -eZ`

**Practice directory this task:** `/etc`

`/etc` is heavily labeled by SELinux, so it is the right directory for context reading.

### 🔁 Warm-Up — Commands from Previous Labs

```bash
cd /tmp/lsZ-lab
getenforce 2>&1 | tee task03-getenforce.txt
ls -l /etc/passwd /etc/shadow > task03-dac.txt 2>&1
grep -n 'passwd' task03-dac.txt
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Read SELinux user, role, type, and level fields on files and processes.

### Main Command Block

```bash
cd /tmp/lsZ-lab
ls -Z /etc/passwd /etc/shadow /etc/ssh/sshd_config 2>&1 | tee task03-file-contexts.txt
id -Z 2>&1 | tee task03-my-context.txt
ps -eZ 2>/dev/null | head -n 8 | tee task03-process-contexts.txt
echo "context lines: $(wc -l < task03-file-contexts.txt)"
grep -E 'passwd|shadow|sshd_config' task03-file-contexts.txt
```

### Human-Readable Breakdown

- `ls -Z` shows file SELinux contexts.
- `id -Z` shows your current SELinux user context.
- `ps -eZ` shows process contexts.
- `2>&1 | tee` captures both success and errors, useful when SELinux tooling is unavailable.
- `grep -E` filters multiple names with one regular expression.

### Reading It Left to Right

`ls -Z /etc/passwd /etc/shadow /etc/ssh/sshd_config 2>&1 | tee task03-file-contexts.txt`

1. `ls -Z` requests SELinux labels.
2. Three `/etc` paths are inspected.
3. `2>&1` merges errors with normal output.
4. `| tee` displays and saves the transcript.

### The Story

SELinux does not care only about users. It mainly cares about *types*. The file type (`passwd_file_t`, `shadow_t`, `sshd_etc_t`, etc.) must match what the policy expects for the process trying to use it.

### Expected Output

```text
system_u:object_r:passwd_file_t:s0 /etc/passwd
system_u:object_r:shadow_t:s0 /etc/shadow
system_u:object_r:sshd_etc_t:s0 /etc/ssh/sshd_config
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

If SELinux is disabled, commands may print `?` or errors. Record that as an environment warning, not a lab failure.

### Switches Table

| Token | Meaning |
|---|---|
| `ls -Z` | Show SELinux context |
| `id -Z` | Show current user's SELinux context |
| `ps -eZ` | Show all process contexts |
| `grep -E` | Extended regular expression |
| `grep -n` | Include line numbers |
| `2>&1 | tee` | Merge errors and save visible transcript |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | SELinux user | First context field |
|  | SELinux role | Second context field |
|  | SELinux type | Third field; most important for RHCSA |
|  | SELinux level | Fourth field (`s0`, MLS/MCS) |
|  | Process context | What a running process is allowed to access |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| T01 | You switch to permissive and hide the real label problem | Keep enforcing; inspect `ls -Z`, `ps -eZ`, and AVC logs before changing mode |

### 🔁 Persistence Check

| What was configured | Verification command | Why it matters |
|---|---|---|
| SELinux current mode | `getenforce` | confirms enforcing/permissive/disabled |
| File contexts visible | `ls -Z /etc/passwd` | confirms label reading works |
| Process contexts visible | `ps -eZ | head` | confirms process labels are visible |

```bash
getenforce 2>/dev/null || echo "getenforce unavailable"
ls -Z /etc/passwd 2>/dev/null || echo "file contexts unavailable"
ps -eZ 2>/dev/null | head -n 3 || echo "process contexts unavailable"
```

### 📓 Journal Write — Before Cleanup

```bash
LAB=lab06
TASK=task3
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"
cat > "$JDIR/done.txt" <<EOF
LAB:    lab06
TASK:   task3
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname)
STATUS: COMPLETE
EOF
cat > "$JDIR/notes.txt" <<EOF
TOPIC:    SELinux contexts on files, users, and processes
COMMANDS: ls -Z, id -Z, ps -eZ, getenforce, grep -E
TRAPS:    T01
MISSED:   note SELinux disabled if getenforce says Disabled
NEXT:     task4 — expected contexts and restorecon dry run
EOF
ls -la "$JDIR"
echo "exit was: $?"
```

### 🧹 Cleanup

```bash
rm -f /tmp/lsZ-lab/task03-getenforce.txt /tmp/lsZ-lab/task03-dac.txt /tmp/lsZ-lab/task03-file-contexts.txt /tmp/lsZ-lab/task03-my-context.txt /tmp/lsZ-lab/task03-process-contexts.txt
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `ls -Z` shows `?` | SELinux may be disabled; run `getenforce` |
| `ps -eZ` fails | SELinux tools/policy not available; record environment warning |
| `id -Z` says not enabled | Practice DAC tasks now; redo SELinux tasks on enforcing host |

> **STOP — paste output before Task 4.**

---

## Task 4 — Compare Actual and Expected Contexts Safely

**Practice directory this task:** `/etc`

SELinux fixes require the right order: define expected context, apply it with `restorecon`, then verify.

### 🔁 Warm-Up — Commands from Previous Labs

```bash
cd /tmp/lsZ-lab
set -o noclobber
ls -Z /etc/passwd > task04-context.txt 2>&1
echo "second write" > task04-context.txt 2>/dev/null || echo "noclobber blocked overwrite"
ls -Z /etc/group >| task04-context.txt 2>&1
set +o noclobber
tail -n 1 task04-context.txt
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Practice the safe SELinux verification flow without changing system policy: inspect expected contexts, dry-run `restorecon`, and rehearse the `semanage` → `restorecon` order.

### Main Command Block

```bash
cd /tmp/lsZ-lab
matchpathcon /etc/passwd /etc/shadow /etc/ssh/sshd_config 2>&1 | tee task04-expected.txt
restorecon -nRv /etc/ssh 2>&1 | head -n 10 | tee task04-restorecon-dryrun.txt
semanage fcontext -l 2>/dev/null | grep -E '/etc/(passwd|shadow|ssh)' | head -n 10 | tee task04-fcontext-list.txt || echo "semanage unavailable"
echo "expected lines: $(wc -l < task04-expected.txt)"
```

### Human-Readable Breakdown

- `matchpathcon` shows the context SELinux expects for a path.
- `restorecon -nRv` dry-runs a recursive relabel and prints what it *would* do.
- `semanage fcontext -l` lists persistent file-context rules.
- `grep -E` filters for important `/etc` paths.
- No system file is modified because `restorecon` uses `-n` (no change).

### Reading It Left to Right

`restorecon -nRv /etc/ssh 2>&1 | head -n 10 | tee task04-restorecon-dryrun.txt`

1. `restorecon` checks SELinux contexts.
2. `-n` means no changes.
3. `-R` means recursive.
4. `-v` means verbose.
5. `/etc/ssh` is the target.
6. Output is merged, limited, displayed, and saved.

### The Story

The correct persistent SELinux fix is not “set permissive.” It is: add or confirm the fcontext rule, run `restorecon -Rv`, then verify with `ls -Z`. Task 4 rehearses that order safely.

### Expected Output

```text
/etc/passwd system_u:object_r:passwd_file_t:s0
/etc/shadow system_u:object_r:shadow_t:s0
/etc/ssh/sshd_config system_u:object_r:sshd_etc_t:s0
expected lines: 3
```

`restorecon -nRv` may print nothing if everything is already correct.

### Switches Table

| Token | Meaning |
|---|---|
| `matchpathcon` | Show expected SELinux context |
| `restorecon -n` | No changes; dry run |
| `restorecon -R` | Recursive |
| `restorecon -v` | Verbose |
| `semanage fcontext -l` | List persistent file-context rules |
| `set -o noclobber` | Prevent accidental overwrite |
| `>|` | Force overwrite intentionally |
| `tail -n 1` | Last line only |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | Expected context | Policy's correct label for a path |
|  | Actual context | What `ls -Z` currently shows |
|  | `restorecon -nRv` | Dry-run relabel recursively and verbosely |
|  | `semanage fcontext -l` | Persistent context rules |
|  | `noclobber` | Protects evidence files from overwrite |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| T02 | You add an fcontext rule but forget `restorecon -Rv` | Always run `restorecon -Rv PATH`, then verify with `ls -Z PATH` |

### 🔁 Persistence Check

| What was configured | Verification command | Why it matters |
|---|---|---|
| Persistent fcontext rules | `semanage fcontext -l | grep PATH` | confirms policy store rule exists |
| Applied label | `ls -Z PATH` | confirms inode label changed |
| Dry-run safety | `restorecon -nRv /etc/ssh` | proves what would change before changing it |

```bash
semanage fcontext -l 2>/dev/null | grep -E '/etc/(passwd|shadow|ssh)' | head || echo "semanage unavailable"
restorecon -nRv /etc/ssh 2>&1 | head
ls -Z /etc/ssh/sshd_config 2>/dev/null || echo "context unavailable"
```

### 📓 Journal Write — Before Cleanup

```bash
LAB=lab06
TASK=task4
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"
cat > "$JDIR/done.txt" <<EOF
LAB:    lab06
TASK:   task4
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname)
STATUS: COMPLETE
EOF
cat > "$JDIR/notes.txt" <<EOF
TOPIC:    Expected SELinux contexts and restorecon dry run
COMMANDS: matchpathcon, restorecon -nRv, semanage fcontext -l, ls -Z
TRAPS:    T02
MISSED:   none
NEXT:     task5 — capstone audit with ls -lZ, ps -eZ, journal index
EOF
ls -la "$JDIR"
echo "exit was: $?"
```

### 🧹 Cleanup

```bash
rm -f /tmp/lsZ-lab/task04-context.txt /tmp/lsZ-lab/task04-expected.txt /tmp/lsZ-lab/task04-restorecon-dryrun.txt /tmp/lsZ-lab/task04-fcontext-list.txt
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `matchpathcon: command not found` | Install policycoreutils tools or use `ls -Z` for now |
| `semanage: command not found` | Install `policycoreutils-python-utils` on RHEL/Rocky |
| `restorecon` wants to change many files | You used a broad target; narrow the path before real relabeling |

> **STOP — paste output before Task 5.**

---

## Task 5 — Capstone: Audit `/etc` with DAC and SELinux Evidence

**Practice directory this task:** `/etc`

This capstone combines `ls -l`, `ls -Z`, process contexts, expected-context checks, persistence verification, and final cleanup.

### 🔁 Warm-Up — Commands from Previous Labs

```bash
cd /tmp/lsZ-lab
false | true
echo "PIPESTATUS=${PIPESTATUS[@]}"
journalctl --no-pager -n 3 2>/dev/null | tee task05-journal-sample.txt
ls /proc/self/fd | sort -n | tee task05-fds.txt
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Produce a compact audit bundle proving you can read `/etc` file metadata and SELinux context evidence.

### Main Command Block

```bash
cd /tmp/lsZ-lab
ls -lZ /etc/passwd /etc/group /etc/ssh/sshd_config 2>&1 | tee task05-file-audit.txt
ps -eZ 2>/dev/null | head -n 10 | tee task05-process-audit.txt || echo "process contexts unavailable"
matchpathcon /etc/passwd /etc/group /etc/ssh/sshd_config 2>&1 | tee task05-expected-audit.txt || echo "matchpathcon unavailable"
echo "file audit lines: $(wc -l < task05-file-audit.txt)"
echo "process audit lines: $(wc -l < task05-process-audit.txt 2>/dev/null || echo 0)"
find /root/rhcsa_journal/lab06 -name done.txt 2>/dev/null | sort
```

### Human-Readable Breakdown

- `ls -lZ` combines DAC metadata and SELinux context in one view.
- `ps -eZ` connects process context to file context thinking.
- `matchpathcon` shows expected labels.
- `wc -l < file` proves evidence files have content.
- `find /root/rhcsa_journal/lab06 -name done.txt` proves progress is resumable.

### Reading It Left to Right

`ls -lZ /etc/passwd /etc/group /etc/ssh/sshd_config 2>&1 | tee task05-file-audit.txt`

1. `ls` lists files.
2. `-l` includes DAC metadata.
3. `-Z` includes SELinux context.
4. `/etc` files are targets.
5. `2>&1` merges errors.
6. `tee` saves and displays.

### The Story

When a service fails, your first audit is simple: what are the Unix permissions, who owns the file, what SELinux type does it have, and what process type is trying to access it? This capstone builds that forensic reflex.

### Expected Output

```text
-rw-r--r--. root root system_u:object_r:passwd_file_t:s0 /etc/passwd
-rw-r--r--. root root system_u:object_r:etc_t:s0 /etc/group
-rw-------. root root system_u:object_r:sshd_etc_t:s0 /etc/ssh/sshd_config
file audit lines: 3
/root/rhcsa_journal/lab06/task1/done.txt
...
```

Exact contexts vary slightly by distro and SELinux state.

### Switches Table

| Token | Meaning |
|---|---|
| `ls -lZ` | Long listing plus SELinux context |
| `ps -eZ` | Process list plus SELinux context |
| `matchpathcon` | Expected context lookup |
| `journalctl --no-pager -n 3` | Last three journal lines, no pager |
| `/proc/self/fd` | Current process file descriptors |
| `sort -n` | Numeric sort |
| `find -name done.txt` | Locate journal completion markers |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `ls -lZ` | DAC and MAC evidence in one command |
|  | `ps -eZ` | Process SELinux type evidence |
|  | `matchpathcon` | Expected file label |
|  | `done.txt` journal | Reboot-safe completion marker |
|  | `/proc/self/fd` | Open descriptors for current shell process |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| T03 | You change a SELinux boolean without `-P` and it vanishes after reboot | Use `setsebool -P BOOLEAN on`, then verify with `getsebool BOOLEAN` |

### 🔁 Persistence Check

| What was configured | Verification command | Why it matters |
|---|---|---|
| Journal entries | `find /root/rhcsa_journal/lab06 -name done.txt | sort` | confirms resumable completion |
| SELinux boolean persistence habit | `getsebool -a | head` | confirms boolean inspection command works |
| Any real boolean change | `setsebool -P NAME on` then `getsebool NAME` | `-P` is what survives reboot |

```bash
find /root/rhcsa_journal/lab06 -name done.txt | sort
getsebool -a 2>/dev/null | head || echo "getsebool unavailable"
```

### 📓 Journal Write — Before Cleanup

```bash
LAB=lab06
TASK=task5
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"
cat > "$JDIR/done.txt" <<EOF
LAB:    lab06
TASK:   task5
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname)
STATUS: COMPLETE
EOF
cat > "$JDIR/notes.txt" <<EOF
TOPIC:    Capstone audit with ls -lZ, ps -eZ, matchpathcon, journal index
COMMANDS: ls -lZ, ps -eZ, matchpathcon, journalctl --no-pager, find done.txt, getsebool
TRAPS:    T03
MISSED:   none
NEXT:     lab07 — touch timestamps (5-task format)
EOF
find /root/rhcsa_journal/lab06 -name done.txt | sort
echo "exit was: $?"
```

### 🧹 Cleanup

```bash
cd /tmp
rm -rf /tmp/lsZ-lab
echo "exit was: $?"
exit
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `getsebool` unavailable | Install SELinux tools or redo on RHEL/Rocky |
| `process audit lines: 0` | SELinux disabled; record environment warning |
| `sshd_config` missing | Use another `/etc/ssh/*` file from `ls /etc/ssh` |

> **STOP — paste output. Lab 06 complete.**

---

## Checklist (5 Tasks — Permanent)

- [ ] Task 1 — read `ls -l` columns on `/etc`
- [ ] Task 2 — identify file types and permission characters
- [ ] Task 3 — read SELinux contexts with `ls -Z`, `id -Z`, `ps -eZ`
- [ ] Task 4 — compare actual and expected contexts safely
- [ ] Task 5 — capstone audit with DAC + SELinux evidence

---

## Trap Rotation

| Trap | Rehearsed where |
|---|---|
| T01 | Tasks 1–3 — inspect evidence instead of setting permissive |
| T02 | Task 4 — `semanage fcontext` requires `restorecon -Rv` |
| T03 | Task 5 — `setsebool` needs `-P` for persistence |

Next lab should rotate to a different category, such as fstab/mount traps T07–T10.

---

## Resume Command

```bash
find /root/rhcsa_journal -name done.txt | sort | tail -5
```

---

## Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory Navigation | Gives the `pwd`, `cd`, and `ls` foundation |
| Lab 07 — Timestamps with `touch` | Creates files whose metadata you inspect here |
| Lab 40 — Standard File Permissions | Changes the DAC bits decoded in this lab |
| Lab 47 — ACL Support | Explains `+` in the long-listing mode string |
| Lab 78 — Managing SELinux Modes | Deepens `getenforce` and `setenforce` |

---

## Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical)
