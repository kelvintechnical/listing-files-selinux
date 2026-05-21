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

**Human-Readable Breakdown**

> "Hey bash, make a folder called `listing-lab` inside my home directory — and don't yell at me if it already exists. Then walk into it, then confirm I'm actually there."

**Reading it left to right:**
- `mkdir` → "**m**a**k**e **dir**ectory."
- `-p` → "**p**arents flag — if any directories in the path don't exist, create them too. And if the final folder already exists, don't crash."
- `~/listing-lab` → "`~` expands to my home directory. So this is `/home/ec2-user/listing-lab`."
- `cd ~/listing-lab` → "step into the folder I just made."
- `pwd` → "**p**rint **w**orking **d**irectory — verify I landed in the right place."

**The story:** Every good lab starts in a clean, dedicated workspace. `mkdir -p` is the "make this folder, no drama" idiom — safe to run twice, safe to run on a missing parent path. The `cd && pwd` pattern is the universal "move there, confirm I'm there" reflex.

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

**Human-Readable Breakdown**

> "Make me two empty files. Make me a folder. Make me a symbolic link that points at one of the files. Then show me everything so I can see all four file types in one shot."

**Reading it left to right:**
- `touch file_a.txt file_b.log` → "create two empty regular files. `touch` normally updates a file's timestamp; if the file doesn't exist, it creates an empty one."
- `mkdir subdir` → "**m**a**k**e a **dir**ectory called `subdir`."
- `ln -s file_a.txt link_a` → "**l**i**n**k, `-s` = **s**ymbolic. Create a symlink named `link_a` whose target is `file_a.txt`."
- `ls` → "show me what's in here now."

**The story:** Linux treats nearly everything as a file, but there are different *types* of files. This task seeds the workspace with one of each common type so the rest of the lab can demonstrate how `ls` distinguishes them. The plain `ls` output looks deceptively uniform — that's the point. The next task uses `ls -l` to reveal the differences.

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

**Human-Readable Breakdown**

> "Show me the detailed listing of everything in this folder — one row per file — and put a single letter at the very start of each row that tells me what *type* of thing it is."

**Reading it left to right:**
- `ls` → "list."
- `-l` → "**l**ong format — one entry per line, every column of metadata visible."

**Focus on the first character of each row:**
- `-` (dash) → "regular file"
- `d` → "directory"
- `l` → "symbolic link"
- `c` → "character device (e.g., terminal)"
- `b` → "block device (e.g., disk)"
- `s` → "socket"
- `p` → "named pipe (FIFO)"

**The story:** The very first character of a long listing answers "what kind of file is this?" before you read anything else. Once that letter is committed to muscle memory, you'll scan a directory in milliseconds. When a symlink shows `lrwxrwxrwx`, don't panic about the wide-open permissions — symlink permissions are always ignored. What matters is the **target's** permissions, not the link's.

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

**Human-Readable Breakdown**

> "Set the permissions on three things using three-digit octal numbers. Set `file_a.txt` to `750` (owner full access, group can read+execute, others get nothing). Set `file_b.log` to `644` (owner can read+write, everyone else can read). Set `subdir` to `700` (only the owner can do anything). Then verify by showing me the long listing."

**Reading it left to right:**
- `chmod` → "**ch**ange **mod**e — change a file's permission bits."
- `750` → "three octal digits. Each digit is a trio of `rwx` bits. Read it as: **owner = 7**, **group = 5**, **other = 0**."
- `7 = rwx` → "read + write + execute (4 + 2 + 1)"
- `5 = r-x` → "read + execute (4 + 1)"
- `4 = r--` → "read only"
- `0 = ---` → "nothing"
- `ls -l file_a.txt file_b.log subdir` → "verify by re-listing each one."

**The story:** Octal permissions are how every sysadmin and exam graders speak. `750` isn't three random digits — it's three answers to "what can owner do?", "what can group do?", "what can everyone else do?" Each digit is a sum: 4 (read) + 2 (write) + 1 (execute). Memorize 7 (full), 5 (read+execute, common for directories), 4 (read-only), 6 (read+write, common for config files), 0 (nothing). On directories, the bits mean slightly different things: `x` becomes "can I `cd` into it?" rather than "can I run it?"

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

**Human-Readable Breakdown**

> "Show me the long listing of `file_a.txt` and pay close attention to the 11th character — the symbol that sits right after the 9 permission bits. That one tiny character tells me whether SELinux or extra ACLs are in play."

**Reading it left to right:**
- `ls -l file_a.txt` → "long listing of just this one file."
- Count to character **11** in the leftmost column. After `-rwxr-x---` comes... a single character.
- `.` (dot) → "this file has an **SELinux** context. SELinux is enforcing."
- `+` (plus) → "this file has **extra POSIX ACL entries** beyond owner/group/other (see Task 6)."
- (nothing) → "neither SELinux nor ACL — usually a filesystem mounted without those features."

**The story:** That tiny 11th character is one of the most overlooked symbols in Linux. Beginners ignore it. Senior admins use it as a free heads-up: "SELinux is active here" or "this file has extra ACLs I need to inspect with `getfacl`." On modern RHEL/CentOS/Fedora, you'll see `.` everywhere because SELinux is on by default. On a fresh NFS or vfat mount, that column will be blank.

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

**Human-Readable Breakdown**

> "Add an extra permission rule to `file_a.txt` that gives the user `nobody` read+execute access. Then re-run the long listing to see the new `+` indicator pop up. Then dump the full ACL so I can see every entry."

**Reading it left to right:**
- `setfacl` → "**set** **f**ile **ACL** — add or change Access Control List entries (extra permissions beyond owner/group/other)."
- `-m` → "**m**odify the ACL (add/change entries; don't replace the whole list)."
- `u:nobody:rx` → "ACL entry: for **u**ser `nobody`, grant `r` (read) and `x` (execute)."
- `file_a.txt` → "the target file."
- `ls -l file_a.txt` → "long listing — look at character 11 for the new `+`."
- `getfacl file_a.txt` → "**get** **f**ile **ACL** — dump every ACL entry on the file."

**The story:** Standard Unix permissions only have three slots: owner, group, other. ACLs let you add named exceptions — "give this specific user/group these specific bits." When you add even one ACL entry, `ls -l` changes the 11th character from `.` to `+` so you know to use `getfacl` to see the full picture. ACLs are how you grant precise access without changing the file's owner or group.

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

**Human-Readable Breakdown**

> "Fill `file_b.log` with 2 megabytes of pure zeros using `dd`, throw away any error messages from `dd`, then show me the long listing of the file with sizes in human-readable units (K/M/G instead of raw bytes)."

**Reading it left to right (the `dd` command):**
- `dd` → "**d**isk **d**ump (or **d**ata **d**uplicator) — copy raw bytes from one place to another."
- `if=/dev/zero` → "**i**nput **f**ile = `/dev/zero` (a special device that produces an endless stream of zero bytes)."
- `of=file_b.log` → "**o**utput **f**ile = `file_b.log` (overwrite it)."
- `bs=1M` → "**b**lock **s**ize = 1 mebibyte (1024 × 1024 = 1,048,576 bytes)."
- `count=2` → "write 2 blocks total. So: 2 × 1 MiB = 2 MiB written."
- `2>/dev/null` → "redirect `dd`'s status messages (stderr, port 2) into the trash so we don't see them."

**Reading it left to right (the `ls` command):**
- `ls -lh file_b.log` → "**l**ong listing, **h**uman-readable sizes."
- Result: `2.0M` instead of `2097152`.

**The story:** `dd` here is just a quick way to manufacture a file of a known size so the human-readable column has something interesting to show. `-h` converts raw byte counts into the units humans actually use — `K` (kilobyte = 1024 bytes), `M` (mebibyte = 1024 KiB), `G` (gibibyte = 1024 MiB). It only works in combination with `-l` or `-s`, because plain `ls` doesn't show sizes at all.

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

**Human-Readable Breakdown**

> "Show me the metadata of `subdir` itself — its permissions, owner, group, mtime — but do NOT descend into it. Same for `/etc`."

**Reading it left to right:**
- `ls` → "list."
- `-l` → "long format."
- `-d` → "**d**irectory entry — describe the directory itself, don't dump its contents."
- `subdir /etc` → "two directories to inspect."

**The story:** Without `-d`, `ls -l /etc` walks **into** `/etc` and prints hundreds of lines. With `-d`, it stops at the door and shows you `/etc`'s own permissions and ownership. This is **the** flag for verifying "did I create this directory with mode 0755?" on every RHCSA permission task. Always pair `chmod`/`chown` on a directory with `ls -ld` to verify.

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

**Human-Readable Breakdown**

> "Show me the long listing of this folder — but include hidden dotfiles AND the `.` (this directory) and `..` (parent directory) entries that normally stay invisible."

**Reading it left to right:**
- `ls` → "list."
- `-l` → "long format."
- `-a` → "**a**ll — show everything: dotfiles, `.`, and `..`."

**The story:** Combining flags is a Unix superpower. `-la` is just `-l` and `-a` glued together — you can stack flags in any order: `-la`, `-al`, `-l -a` all mean the same thing. The first row (`.`) shows the current directory's own metadata, and the second row (`..`) shows the parent's. Watch for the `+` on `file_a.txt` — the ACL we added in Task 6 is still there.

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

**Human-Readable Breakdown**

> "Show me the SELinux security label on `file_a.txt` — the four-part security tag that policy uses to decide who is allowed to touch this file."

**Reading it left to right:**
- `ls` → "list."
- `-Z` → "show the SELinux context column (capital Z — lowercase `-z` does something different in some tools)."
- The output is **four fields separated by colons**: `user:role:type:level`.

**Reading the context `unconfined_u:object_r:user_home_t:s0`:**
- `unconfined_u` → "SELinux **user** — 'unconfined' means no extra restrictions."
- `object_r` → "SELinux **role** — almost always `object_r` for files."
- `user_home_t` → "SELinux **type** — the field policy actually checks. Files in `/home/*` get `user_home_t`."
- `s0` → "MLS/MCS **level** — sensitivity label. `s0` = default."

**The story:** SELinux adds a security layer **on top of** classic Unix permissions. Even if `chmod 777` says "anyone can do anything," SELinux can still block access if the file's **type** doesn't match what the requesting process's **domain** is allowed to touch. The **type** field is what matters 99% of the time — when Apache returns 403 Forbidden on a file with mode 644, it's almost always because the type is `user_home_t` instead of `httpd_sys_content_t`.

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

**Human-Readable Breakdown**

> "Give me the full long listing AND the SELinux context, both on the same line. One row, every column, every security label."

**Reading it left to right:**
- `ls` → "list."
- `-l` → "long format — all DAC metadata (type, perms, owner, group, size, mtime, name)."
- `-Z` → "additionally show the SELinux context (MAC label)."

**The story:** `-lZ` is the unified security view. In one line you can answer both "who can read this under classic Unix rules?" (DAC) and "what SELinux policy applies to it?" (MAC). When troubleshooting a service that "should work but doesn't," `ls -lZ` is the first command to run because it shows both layers simultaneously.

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

**Human-Readable Breakdown**

> "Show me the SELinux contexts of these five well-known system directories — the directories themselves, not their contents — so I can compare the different default types policy assigns to each location."

**Reading it left to right:**
- `ls` → "list."
- `-Z` → "show SELinux context."
- `-d` → "describe the directories themselves, not their contents."
- The five paths → "Apache root, SSH config, home, temp, system logs — each lives in a different policy zone."

**The story:** Linux's SELinux policy ships with **expected types** for well-known paths. Apache only reads files of type `httpd_sys_content_t`. rsyslog only writes to files of type `var_log_t`. When you copy a file *into* one of these locations, it doesn't automatically get the right type — it keeps the type of wherever it came from. That's the #1 cause of "I copied my web page to `/var/www/html` and Apache returns 403." The fix (covered in Lab 16) is `restorecon` or `chcon`, but you first have to **see** the mismatch with `ls -Zd`.

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

**Human-Readable Breakdown**

> "Show me the SELinux domain that the `sshd` daemon is running under — just the first match. Then show me the domain *my own* shell is running under."

**Reading it left to right (first command):**
- `ps` → "**p**rocess **s**tatus — list processes."
- `-e` → "**e**very process (system-wide, not just mine)."
- `-Z` → "include the SELinux context column."
- `|` → "pipe — feed the output into the next command."
- `grep sshd` → "keep only lines mentioning `sshd`."
- `|` → "pipe again."
- `head -1` → "keep only the first matching line."

**Reading it left to right (second command):**
- `id` → "print identity info."
- `-Z` → "print *only* my current SELinux context (no UID/GID noise)."

**The story:** SELinux is **bidirectional**. Files have **types** (e.g., `httpd_sys_content_t`). Processes have **domains** (e.g., `sshd_t`). Policy is a giant rule set that says "domain X is allowed to read/write/execute files of type Y." For an SSH login to work, the `sshd_t` domain must be allowed to read `etc_t` files (its configs) and `user_home_t` files (your `~/.ssh/authorized_keys`). If policy doesn't allow it, the connection silently fails — and the only way to diagnose it is to look at both sides with `ps -eZ` and `ls -Z`.

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

**Human-Readable Breakdown**

> "Look up what SELinux context these two paths *should* have according to the system's policy database. Don't tell me what they actually have right now — tell me what they're *supposed* to have."

**Reading it left to right:**
- `matchpathcon` → "**match** **path** **con**text — query the policy database for the expected context of a given path."
- `/var/www/html/index.html` → "policy says this should be `httpd_sys_content_t`."
- `/home/ec2-user/.bashrc` → "policy says this should be `user_home_t`."

**The story:** SELinux ships with a giant database (`file_contexts`) that maps path patterns to expected types. `matchpathcon` is your "what's it supposed to be?" lookup tool. Pair it with `ls -Z` to compare expected vs actual. If they differ, the fix is `restorecon` (re-apply the policy-expected context) or `chcon` (manually set a context). On the RHCSA, this comparison is the troubleshooting workflow for any service that won't start because of file labels.

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

**Human-Readable Breakdown**

> "Give me **everything** you know about `file_a.txt` — size, blocks, device, inode, hard links, mode (both octal and symbolic), owner, group, SELinux context, plus all four timestamps."

**Reading it left to right:**
- `stat` → "**stat**istics — dump every piece of metadata on this file."
- `file_a.txt` → "the target."

**Reading the output, line by line:**
- `Size:` → "how many bytes the file contains."
- `Blocks:` → "how many disk blocks the file actually occupies."
- `IO Block:` → "filesystem's preferred I/O size."
- `Device:` → "major,minor numbers of the device the file lives on."
- `Inode:` → "the file's unique disk-level fingerprint on this filesystem."
- `Links:` → "hard link count — how many filenames point to this inode."
- `Access:` (mode line) → "permissions in octal (e.g., `0750`) and symbolic (`-rwxr-x---`)."
- `Uid:` / `Gid:` → "owner / group in both numeric and named form."
- `Context:` → "SELinux context."
- `Access:` (timestamp) → "**atime** — last time the file was read."
- `Modify:` → "**mtime** — last time the file's contents changed."
- `Change:` → "**ctime** — last time the file's metadata (permissions, owner, etc.) changed."
- `Birth:` → "**btime** — when the file was created."

**The story:** `ls -l` shows you one timestamp (mtime). `stat` shows you all four — and that's huge for forensics. "When was this file last *read*?" → atime. "When was its content last changed?" → mtime. "When were the permissions changed?" → ctime. "When was it born?" → btime. Each one answers a different "what happened?" question. The RHCSA exam doesn't quiz on `stat` directly, but RH342 troubleshooting expects you to know these.

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

**Human-Readable Breakdown**

> "Run `stat`, but instead of the full multi-line report, give me a custom one-line summary with just the fields I want, in the order I want."

**Reading it left to right (first command):**
- `stat` → "statistics."
- `-c '...'` → "**c**ustom format string — the quoted text is a template where each `%X` placeholder gets replaced by a specific field."
- `%U %G %a %A %s %n` → "print: owner user, owner group, octal mode, symbolic mode, size in bytes, name."
- `file_a.txt file_b.log` → "run the format on each of these two files."

**Reading it left to right (second command):**
- `stat -c 'name=%n mode=%a context=%C' file_a.txt` → "use literal text mixed with placeholders. Output: `name=file_a.txt mode=750 context=...`. Perfect for log/audit lines."

**Key format placeholders:**
- `%n` → "file **n**ame"
- `%s` → "**s**ize in bytes"
- `%a` → "mode in oct**a**l (e.g., `750`)"
- `%A` → "mode in symbolic form (e.g., `-rwxr-x---`)"
- `%U` → "owner **U**ser name" / `%u` → "**u**ser ID number"
- `%G` → "owner **G**roup name" / `%g` → "**g**roup ID number"
- `%i` → "**i**node"
- `%h` → "**h**ard link count"
- `%C` → "SELinux **C**ontext"
- `%x %y %z %w` → "atime, mtime, ctime, btime"

**The story:** `stat -c` is the script-friendly twin of `stat`. Instead of 10 lines per file, you get exactly the fields you specify on one line. This is **the** RHCSA audit one-liner — `stat -c '%U %G %a %C %n' /var/www/html` answers "owner, group, mode, SELinux context, name" in a single grep-able row. Memorize the placeholders for `%U %G %a %C %n` and you can audit anything fast.

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

**Human-Readable Breakdown**

> "Show me the long listing of these two files — `file_a.txt` and the symlink `link_a` that points at it — and put the inode number (the file's unique disk-level fingerprint) at the very start of each row."

**Reading it left to right:**
- `ls` → "list."
- `-l` → "long format."
- `-i` → "**i**node — prepend each row with the file's inode number."
- `file_a.txt link_a` → "the regular file and the symlink that points at it."

**The story:** Every file in Linux has two identities — a **name** (what you call it) and an **inode** (a unique disk-level number). Two filenames pointing to the same inode are **hard links** — they're literally the same file with two names. A **symlink** is its own separate file containing a path string; it has its own inode, different from the target's. The output of this task makes that visible: `file_a.txt` has inode `1048577`, but `link_a` has inode `1048578` — proving it's a separate file (a symlink), not a hard link.

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

**Human-Readable Breakdown**

> "Run as root, walk into every subdirectory of `/etc/ssh`, and print the SELinux context of every single file you find — so I can verify they all have the correct type."

**Reading it left to right:**
- `sudo` → "**s**uperuser **do** — run the next command as root (needed because some files under `/etc/ssh` are root-only readable)."
- `ls` → "list."
- `-R` → "**R**ecursive — descend into every subdirectory."
- `-Z` → "show SELinux context."
- `/etc/ssh` → "starting point of the walk."

**The story:** After running `restorecon -R` to fix SELinux contexts, you need to verify the fix actually worked across every file in the tree. `ls -RZ` is that verification. Every file under `/etc/ssh` should have the type `etc_t` — if even one file shows a different type (like `default_t` or `user_home_t`), that's the bug. `-R` is safe here because `/etc/ssh` is small; never run `ls -R` on `/` or `/var`.

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

**Human-Readable Breakdown**

> "First command: starting from here, find every regular file where the owner has write permission, and for each match, run `ls -lZ` on it. Second command: starting from here, find every symbolic link and run `ls -lZ` on it."

**Reading it left to right (first command):**
- `find` → "search the filesystem."
- `.` → "starting from the current directory."
- `-type f` → "only regular files (not directories, not symlinks)."
- `-perm /u+w` → "filter by **perm**ission. The `/` means *any of these bits set*. `u+w` = the owner's write bit. So: only files where owner has write."
- `-exec ls -lZ {} \;` → "for each match, run this command. `{}` is replaced with the path of the matched file. `\;` (backslash-semicolon) is the literal end-of-command marker `find` requires."

**Reading it left to right (second command):**
- `find . -type l` → "find symbolic links only."
- `-exec ls -lZ {} \;` → "same template — run `ls -lZ` on each one."

**The story:** `find` answers "which files match this criteria?" `ls -l` answers "what's the metadata for this file?" Together they let you ask compound questions like "show me the long listing of every writable file in this directory." The `-exec` trick — `... -exec CMD {} \;` — is how `find` chains into other tools. `{}` is the placeholder for the matched filename, and `\;` tells `find` "the command ends here." If you forget the `\;`, `find` complains with `missing argument to -exec`.

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

**Human-Readable Breakdown**

> "Audit `/var/www/html` three different ways. First: long listing of the directory itself with its SELinux context. Second: a compact one-line audit using `stat` showing owner, group, mode, context, and name. Third: ask the policy database what context this path is *supposed* to have. All three answers should agree."

**Reading it left to right (line 1 — visual confirmation):**
- `sudo ls -ldZ /var/www/html` → "run as root, long listing, directory itself only (`-d`), with SELinux context (`-Z`)."
- This gives the most readable human view: type, perms, owner, group, context, size, date, name.

**Reading it left to right (line 2 — script-friendly audit):**
- `stat -c '%U %G %a %C %n' /var/www/html` → "custom one-line stat."
- `%U` → "owning **U**ser name"
- `%G` → "owning **G**roup name"
- `%a` → "mode in oct**a**l"
- `%C` → "SELinux **C**ontext"
- `%n` → "file **n**ame"
- Result: `root root 755 system_u:object_r:httpd_sys_content_t:s0 /var/www/html` — perfect for a logbook entry.

**Reading it left to right (line 3 — policy expectation):**
- `matchpathcon /var/www/html` → "look up what policy *says* this path should be labeled as."
- If it matches `ls -Z`'s output → labels are correct. If it doesn't → run `restorecon` to fix.

**The story:** This is the canonical RHCSA verification workflow. The exam doesn't say "use these specific commands" — it says *"confirm `/var/www/html` is owned by `root:root`, mode 0755, with SELinux type `httpd_sys_content_t`."* That single sentence translates into the three-line audit above. After you do it a dozen times, it becomes one keystroke combo your fingers run automatically. Every file/permission task on the exam ends with this verification.

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
