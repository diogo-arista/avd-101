# 03 — Linux & Bash Essentials for Automation

> **Goal of this chapter:** give you enough Linux command-line literacy to navigate, edit, search, pipe, and script around the tools we'll use in every later chapter. Not a Linux course — a *survival kit* aimed at someone who's already comfortable in Cisco/Arista CLI but rarely lives in a Linux shell.

This chapter happens **inside the OrbStack `avdlab` machine** unless noted. Open VS Code Remote-SSH to `avdlab@orb`, then `` Ctrl+` `` for the integrated terminal.

## 1. Why a network engineer needs Linux

You already know one CLI fluently — EOS. Linux is just another CLI with different verbs and a much bigger toolbox. Every tool we'll use lives in or talks to Linux:

- Ansible runs on Linux.
- containerlab orchestrates Linux network namespaces.
- AVD's collection, Python, Jinja, Git — all assume Linux.
- CI runners (GitHub Actions, GitLab) are Linux containers.
- Even your switch — EOS itself — is a Linux distribution under the hood (the `bash` command at `enable` mode drops you into a real Linux shell).

The good news: you only need **maybe 30 commands** and a handful of patterns. We'll cover those.

## 2. Filesystem layout in 60 seconds

| Path | What lives there | Network-engineer analogy |
|---|---|---|
| `/` | Root of everything. | `/` on EOS flash. |
| `/home/<user>` (or `~`) | Your home directory. | The user-specific config tree. |
| `/etc` | System-wide configs. | `running-config` plus everything in `flash:/persist/`. |
| `/var/log` | Logs. | The `logging` buffer + accounting/syslog. |
| `/usr/bin`, `/usr/local/bin` | Most CLI tools live here. | Built-in EOS commands. |
| `/tmp` | Scratch space, wiped on reboot. | RAM-only working buffers. |
| `/opt` | Big optional software packages. | Third-party extensions. |
| `/dev`, `/proc`, `/sys` | Kernel/device pseudo-filesystems. | Hardware probing tables — leave alone unless you know why. |

Useful one-liners:

```bash
pwd            # print working directory
whoami         # current user
hostname       # this machine's name
df -h          # disk usage per filesystem
du -sh ~/      # how big is my home dir
```

## 3. Moving around and looking at files

```bash
cd ~/projects/avd-study     # change directory
cd ..                       # up one
cd -                        # back to previous directory

ls                          # list files
ls -la                      # long format, including hidden (dotfiles)
ls -lah                     # plus human-readable sizes

cat 02-lab-setup.md         # dump file to terminal
less 02-lab-setup.md        # pager: arrows to scroll, q to quit, / to search
head -20 README.md          # first 20 lines
tail -20 README.md          # last 20 lines
tail -f /var/log/syslog     # follow a growing log (like `tail` on EOS)
```

⚠ **`cat` etiquette:** `cat` dumps the whole file. For anything over ~100 lines, use `less`. Dumping a 50,000-line config into a terminal will lock it up briefly.

## 4. Creating, copying, moving, deleting

```bash
mkdir -p labs/03-exercises          # -p creates parents as needed
touch myfile.txt                    # create an empty file
cp source.txt dest.txt              # copy
cp -r src_dir/ dest_dir/            # copy recursively (directories)
mv old_name.txt new_name.txt        # rename or move
mv file.txt ~/elsewhere/            # move into a dir
rm file.txt                         # delete file
rm -r somedir/                      # delete a directory and contents
rm -rf somedir/                     # ditto, no prompts (DANGEROUS — see gotcha)
```

⚠ **`rm -rf` is irreversible.** There is no Trash, no undo, no `wr mem` to rollback. Always check `pwd` and the path you're about to delete. `rm -rf /` will happily try to destroy the system if run as root. A common safer alias is `alias rm='rm -i'` but most pros just slow down and double-check.

## 5. Reading and editing files

You have several text-editor options in the VM:

- **VS Code via Remote-SSH** — what we set up in chapter 02. Use this for any file you'll edit more than once.
- **`nano`** — friendly TUI editor for quick edits inside a terminal. Ctrl+O saves, Ctrl+X exits.
- **`vim`** — powerful but has a learning curve. Press `i` to insert, `Esc` then `:wq` to save+quit, `:q!` to quit without saving.

For *reading* a file without editing, prefer `less` (see §3) over opening it in an editor.

## 6. Searching: `grep`, `find`, and friends

These three are the most-used troubleshooting tools in any Linux shop.

### `grep` — search inside file contents

```bash
grep "BGP" running-config.txt           # lines containing "BGP"
grep -i "bgp" running-config.txt        # case-insensitive
grep -n "neighbor" running-config.txt   # show line numbers
grep -r "TODO" .                        # recursively search current dir
grep -v "shutdown" interfaces.cfg       # invert: lines NOT containing "shutdown"
grep -E "^(vlan|interface)" config.cfg  # extended regex: lines starting with vlan or interface
```

### `find` — search for files by name, size, age

```bash
find . -name "*.yml"                    # all .yml files in or below current dir
find . -type f -name "*.cfg"            # only regular files
find /var/log -mtime -1                 # files modified in last 24h
find . -size +10M                       # files larger than 10 MB
```

### Bonus: `rg` (ripgrep) and `fd` — faster modern alternatives

```bash
sudo apt install -y ripgrep fd-find
rg "neighbor"           # like grep -r but 10x faster and ignores .gitignore
fdfind ".yml$"          # like find but with sane defaults
```

> **Why this matters:** AVD will generate hundreds of `.cfg` files. Knowing how to `grep -r "router bgp 65" intended/configs/` to instantly see every device with a particular ASN is the difference between "5 seconds" and "5 minutes."

## 7. Pipes, redirection, and command chaining

Linux's superpower: gluing small tools together.

| Operator | Meaning |
|---|---|
| `\|` | Pipe stdout of left into stdin of right. |
| `>` | Redirect stdout to file (overwrite). |
| `>>` | Redirect stdout to file (append). |
| `2>` | Redirect stderr. |
| `&>` or `2>&1` | Redirect both stdout and stderr. |
| `&&` | Run right command only if left succeeded. |
| `\|\|` | Run right command only if left failed. |
| `;` | Run sequentially, regardless of success. |

Examples:

```bash
# Show every "interface" line in every .cfg, then sort and uniq them
grep -h "^interface" *.cfg | sort -u

# Save a long command's output for later inspection
ansible-inventory --list > inventory_dump.json

# Run tests, on success run the build, on failure show a message
make test && make build || echo "Build skipped: tests failed"

# Count how many files in a tree mention BGP
grep -lr "router bgp" . | wc -l
```

⚠ The pipe symbol `|` is the closest thing Linux has to EOS' `| include` / `| exclude` / `| begin`. Once you internalize that, EOS feels less alien and Linux feels less arcane — they're the same idea.

## 8. Permissions: `chmod`, `chown`, and that `rwx` thing

A typical `ls -l` line:

```
-rwxr-xr-- 1 diogo staff 1234 May 20 14:22 myscript.sh
```

Breakdown:

```
- rwx r-x r--
^ ^   ^   ^
| |   |   └── others: read only
| |   └────── group:  read + execute
| └────────── owner:  read + write + execute
└──────────── file type ('-' = file, 'd' = dir, 'l' = symlink)
```

Common operations:

```bash
chmod +x script.sh        # add execute for all
chmod 755 script.sh       # rwx for owner, rx for group+others (the most common shell script perm)
chmod 600 ~/.ssh/id_rsa   # rw for owner only (required for SSH keys)
chown diogo:diogo file    # change owner and group
```

> **Numeric mode crib:** each digit is a sum of `r=4 w=2 x=1`. So `755` = 4+2+1 / 4+0+1 / 4+0+1 = `rwxr-xr-x`.

## 9. Processes and signals

```bash
ps -ef | grep ansible     # find a running ansible process
top                       # live process viewer; press 'q' to quit
htop                      # nicer version of top (sudo apt install htop)
kill 12345                # graceful stop of PID 12345 (SIGTERM)
kill -9 12345             # force kill (SIGKILL) — last resort
pkill -f containerlab     # kill processes whose command line matches "containerlab"
```

Job control inside one shell:

```bash
sleep 60 &                # start in background
jobs                      # list backgrounded jobs
fg %1                     # bring job 1 to foreground
Ctrl+Z                    # suspend the foreground job
bg                        # resume the suspended job in background
```

## 10. Environment variables and `PATH`

```bash
echo $HOME                # print HOME variable
echo $PATH                # colon-separated list of where to find executables
export AVD_DEBUG=1        # set a variable for this and child processes
unset AVD_DEBUG
env | sort                # list all env vars
```

`PATH` matters: when you type `containerlab`, the shell searches each directory in `$PATH` in order for a matching file. That's why the chapter 02 `newgrp clab_admins` step matters — group changes have to take effect before the shell can resolve permissions on `/usr/bin/containerlab`.

To make a variable persist across shells, add the `export` line to `~/.bashrc` (bash) or `~/.zshrc` (zsh).

## 11. Bash scripting: the 10% that gets you 90% there

A bash script is just a text file of commands with a shebang at the top:

```bash
#!/usr/bin/env bash
set -euo pipefail   # safety: exit on error, undefined var, or pipe failure

DEVICE="$1"         # first argument
echo "Checking ${DEVICE}…"

if ssh -o ConnectTimeout=5 admin@"${DEVICE}" "show version" > /dev/null 2>&1; then
    echo "${DEVICE} OK"
else
    echo "${DEVICE} UNREACHABLE" >&2
    exit 1
fi
```

Make it executable and run it:

```bash
chmod +x check.sh
./check.sh leaf1
```

Key patterns:

- **`set -euo pipefail`** at the top of every script — it turns silent failures into loud ones. Future you will thank you.
- **`$1 $2 …`** for positional args; **`$#`** for the count.
- **`if … then … fi`** with `[[ … ]]` for conditions: `[[ -f file ]]` (file exists), `[[ -z "$x" ]]` (empty), `[[ "$a" == "$b" ]]` (equal).
- **`for x in …; do … done`** for loops.
- **Backticks or `$(…)`** for command substitution: `count=$(ls | wc -l)`.

⚠ Bash is *not* the right tool for anything non-trivial. Once a script grows past ~50 lines, rewrite it in Python (chapter 11). Bash is glue; Python is plumbing.

## 12. Package management on Ubuntu

```bash
sudo apt update                    # refresh package index (don't skip this)
sudo apt install -y htop jq tree   # install named packages
sudo apt remove htop               # uninstall
apt search ripgrep                 # find packages
apt show ripgrep                   # details on a package
```

You'll install dozens of packages over the curriculum. A few you'll want right now:

```bash
sudo apt install -y jq tree ripgrep fd-find tmux htop unzip
```

- `jq` — query/transform JSON from the shell (used everywhere with eAPI).
- `tree` — visual directory tree.
- `tmux` — multiple persistent terminals inside one SSH session.
- `unzip` — for handling release archives.

## 13. SSH and key auth, briefly

```bash
ssh admin@172.20.20.2                                # password auth
ssh -i ~/.ssh/id_ed25519 admin@switch                # specific key
ssh-keygen -t ed25519 -C "you@example.com"           # generate a new keypair
ssh-copy-id admin@switch                             # push your pubkey to a host
scp file.txt admin@switch:/tmp/                      # copy a file
```

For repeated targets, put aliases in `~/.ssh/config`:

```
Host leaf1
    HostName 172.20.20.3
    User admin
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

…then `ssh leaf1` is enough.

> **What's StrictHostKeyChecking?** SSH normally remembers a host's fingerprint and warns loudly if it changes (potential MITM). For *labs* where containers redeploy with new keys constantly, you can disable the check. Never do this against production gear.

## 14. The one-line idioms you'll reuse forever

```bash
# Pretty-print a JSON file
cat data.json | jq .

# Show only specific keys from JSON
curl -sk -u admin:admin https://leaf1/command-api -d @payload.json | jq '.result[0].vrfs'

# Count occurrences of a pattern across files
grep -c "neighbor" *.cfg | sort -t: -k2 -n

# Find files modified in last 10 minutes (great for "what did that ansible run change?")
find . -mmin -10 -type f

# Replace a string across multiple files (always preview with grep first!)
grep -rl "OLD_STRING" .             # which files contain it
sed -i 's/OLD_STRING/NEW_STRING/g' file.txt   # do the replace (Linux sed)

# Show the diff between two configs
diff -u baseline.cfg current.cfg | less

# Sum a column of numbers
awk '{ sum += $3 } END { print sum }' file.txt
```

## 15. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **stdin / stdout / stderr** | The three default streams for any process: input, normal output, error output. |
| **PID** | Process ID — every running process has one (`ps`, `kill`). |
| **TTY** | Terminal device — the channel between you and a shell. |
| **`$?`** | Exit code of the last command. `0` = success, anything else = failure. |
| **glob** | Shell wildcard, e.g. `*.yml`, `intended/configs/*.cfg`. |
| **shebang** | `#!/usr/bin/env bash` on line 1 of a script — tells the OS what interpreter to use. |
| **dotfile** | A file or directory whose name starts with `.` — hidden from `ls` by default. Configs live here. |
| **symlink** | A pointer file: `ln -s /real/path link_name`. |
| **PATH** | Where the shell looks for executables. |

---

## 🧪 Exercise

Inside your OrbStack VM (VS Code Remote-SSH terminal works perfectly):

1. **Navigate** to `/Users/$USER/projects/avd-study` and `ls -lah`. Identify which files are dotfiles vs regular files.
2. **Find every Markdown file** in the project and count the total lines:
   ```bash
   find . -name "*.md" -type f | xargs wc -l
   ```
   Then do it again with `rg --files -g '*.md' | xargs wc -l` and compare.
3. **Pipe practice:** combine commands to count how many times the word "AVD" appears across all chapters:
   ```bash
   grep -ohi "avd" *.md | wc -l
   ```
4. **Write a tiny script** at `~/check_lab.sh` that pings the four lab devices (`172.20.20.2-5`) once each and prints `OK` or `FAIL` per IP. Use the bash patterns from §11. Make it executable with `chmod +x`.
5. **Install `jq`** and run the chapter 02 §9 eAPI curl against `leaf1`, then pipe through `jq .` to pretty-print. Then drill in with `jq '.result[0]'` to see just the inner result.
6. **Optional:** create a `~/.ssh/config` alias for each lab device (spine1, spine2, leaf1, leaf2) using the IPs from `containerlab inspect`. Confirm you can `ssh leaf1` directly.

---

**Next:** [04 — Git & GitHub for Network Engineers](04-git-github.md)
