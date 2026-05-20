# 04 — Git & GitHub for Network Engineers

> **Goal of this chapter:** turn Git from a black box into a tool you trust. By the end you should be able to clone, branch, commit, push, pull, open a PR, review a diff, and recover from common mistakes — all from the command line and from VS Code's Git panel.

You already have a working repo from chapter 02: `https://github.com/diogo-arista/avd-101`. We'll use it as the lab throughout this chapter.

## 1. What Git actually is, in network-engineer terms

Git is a **version control system**: it tracks the history of changes to a folder of files. Think of it as:

- A snapshot recorder. Every `git commit` is a "save point" of your entire project.
- A time machine. You can rewind to any past commit and look around.
- A merge engine. Multiple people can work on the same files; Git stitches changes together.
- An audit log. Every commit has an author, timestamp, and message.

**GitHub** (and GitLab, Bitbucket) is a *hosting service* for Git repos that adds collaboration features — pull requests, issues, CI integration, code review tools. Git works fine without it (you can use Git purely locally), but a remote like GitHub is how teams share work.

> **Why a network engineer should care:** the AVD project repo IS your source of truth. Every network change is a commit. Every change review is a pull request. Every audit is `git log`. If `wr mem` is your today, Git is your tomorrow.

## 2. The mental model: three places your files live

```
┌────────────────────┐   git add    ┌────────────────────┐   git commit   ┌────────────────────┐
│   WORKING TREE     │ ───────────▶ │   STAGING AREA     │ ─────────────▶ │   LOCAL REPO       │
│  (files on disk)   │              │  (next snapshot)   │                │ (commit history)   │
└────────────────────┘              └────────────────────┘                └─────────┬──────────┘
                                                                                    │ git push
                                                                                    ▼
                                                                          ┌────────────────────┐
                                                                          │   REMOTE REPO      │
                                                                          │     (GitHub)       │
                                                                          └────────────────────┘
```

- **Working tree** — the files you see and edit.
- **Staging area** (a.k.a. *index*) — what you've marked for the *next* commit. Lets you commit some changes but not others.
- **Local repo** — the `.git/` directory in your project. Contains the full history.
- **Remote repo** — a copy on GitHub. Multiple "remotes" possible; the default is named `origin`.

The four verbs `add`, `commit`, `push`, `pull` move things between these layers.

## 3. Identity setup (one-time)

Git stamps every commit with `user.name` and `user.email`. Set them globally:

```bash
git config --global user.name "Diogo Mendes"
git config --global user.email "diogo@example.com"
git config --global init.defaultBranch main      # match GitHub default
git config --global pull.rebase false             # default "merge" behaviour on pull
```

Inspect with `git config --global --list`. (We did this already on your Mac in chapter 02 — verify in the VM too if you'll commit from there.)

## 4. Cloning your repo into the VM

You already have the repo on your Mac at `/Users/diogo/projects/avd-study` (shared into the VM at the same path), so a second clone inside the VM isn't strictly necessary. But cloning *is* the entry point for any new contributor, and you should practice it:

```bash
# Inside the OrbStack VM, in a separate location:
cd ~
git clone https://github.com/diogo-arista/avd-101.git
cd avd-101
ls
```

If `gh auth login` hasn't been done inside the VM (only on the Mac, per chapter 02), HTTPS clone of a public repo still works — no auth needed for read. Pushing would need auth (see §11).

## 5. The everyday loop

The 90% workflow you'll use forever:

```bash
git status                  # what changed? what's staged?
git diff                    # unstaged changes (working → staged)
git diff --staged           # staged changes (staged → last commit)

git add path/to/file.md     # stage one file
git add .                   # stage everything in current dir (use carefully)

git commit -m "describe what changed and why"

git push                    # send commits to origin/<current-branch>
git pull                    # fetch + merge from origin/<current-branch>
```

⚠ **Always read `git status` before `git add .`** — that command will sweep up *everything* including test outputs, secrets, and temp files. Better habit: stage individual files, or `git add -p` to review each chunk interactively.

## 6. Branches and the feature-branch workflow

A **branch** is a movable pointer to a commit. `main` (or `master`) is the default. Real work happens on short-lived feature branches.

```bash
git branch                          # list local branches; * marks current
git branch -a                       # include remote-tracking branches
git switch -c feature/add-vlan-200  # create + switch to new branch
git switch main                     # switch back to main
git switch -                        # switch to previous branch (like cd -)
```

The feature-branch workflow:

```
main:           A───B───────────────────E    (merged)
                     \                 /
feature/add-vlan-200: C───D───────────/
```

1. `git switch -c feature/add-vlan-200` from `main`.
2. Edit files, `git add`, `git commit` — possibly several commits.
3. `git push -u origin feature/add-vlan-200` (first push needs `-u` to set tracking).
4. Open a Pull Request on GitHub.
5. Reviewers comment, you push more commits.
6. PR is approved and merged → branch can be deleted.

> **Why feature branches matter for network changes:** the diff in a PR is the unit of network change. Reviewers see *exactly* what config will change. Merging means "this change is approved, ready to deploy."

## 7. Writing useful commit messages

A good commit message has two parts:

```
subject line, ≤72 chars, imperative mood

Longer explanation paragraph if needed. Wrap at ~72 chars.
Explain WHY this change, not WHAT (the diff shows what).
Reference any related issues like #42 or fabric tickets.
```

Examples — bad vs good:

```
❌ "fix"
❌ "Added stuff"
❌ "WIP"

✅ "Add VLAN 200 (VOICE) to DC1 L3 leaves"
✅ "Increase AVD BGP holdtime to 90s after fabric flap on 2026-04-12"
✅ "Refactor group_vars/DC1.yml: extract management subnets into separate file"
```

Read your own commit log from a year ago. Will you understand what each commit did without opening it?

## 8. Reading history and diffs

```bash
git log                             # full log
git log --oneline                   # one line per commit
git log --oneline --graph --all     # ASCII branch graph (very useful)
git log -p path/to/file             # show diffs for each commit touching a file
git log --since="1 week ago"        # time filter
git blame README.md                 # who last modified each line
git show <commit-hash>              # full details of a single commit
```

For richer browsing, the VS Code Source Control panel (`Ctrl+Shift+G`) shows the same info graphically — branches, file history, side-by-side diffs.

## 9. Undoing things (the survival sub-chapter)

This is where Git scares people. Here's the safe set of "undo" operations.

### Unstage a file (`git add` was a mistake)

```bash
git restore --staged file.md         # newer syntax
git reset HEAD file.md               # older syntax, same effect
```

### Discard uncommitted changes in a file (revert to last commit)

```bash
git restore file.md                  # ⚠ destroys uncommitted edits
```

### Amend the last commit (typo in message, or forgot a file)

```bash
git add forgotten_file.md
git commit --amend                   # opens editor for message
git commit --amend --no-edit         # keep message, just add the file
```

⚠ Only amend commits you have **not pushed yet**. Amending a pushed commit rewrites history and breaks anyone who pulled it.

### Revert a pushed commit (safe — creates a new commit that undoes it)

```bash
git revert <commit-hash>             # creates a new "undo" commit
```

### Discard the last *unpushed* commit but keep the changes

```bash
git reset --soft HEAD~1              # uncommit, keep changes staged
git reset HEAD~1                     # uncommit, keep changes unstaged
git reset --hard HEAD~1              # ⚠ uncommit AND wipe changes — destructive
```

### "I'm completely lost, what was I doing?"

```bash
git reflog                           # journal of HEAD movements; you can reset back to any state
```

`git reflog` is the safety net. Almost nothing in Git is truly gone for ~90 days unless you `git gc --prune=now`. If you panic, take a deep breath, run `git reflog`, and you'll see your way back.

## 10. Pulling, fetching, and dealing with remote changes

```bash
git fetch                  # download remote refs/commits, do NOT modify working tree
git pull                   # fetch + merge into current branch
git pull --rebase          # fetch + rebase your local commits on top
```

If `git pull` finds conflicts (you changed line 10, someone else also changed line 10):

```
CONFLICT (content): Merge conflict in group_vars/DC1.yml
Automatic merge failed; fix conflicts and then commit the result.
```

Open the conflicted file — you'll see Git's markers:

```
<<<<<<< HEAD
   asn: 65001
=======
   asn: 65002
>>>>>>> origin/main
```

Edit to the correct content, remove the markers, then:

```bash
git add group_vars/DC1.yml
git commit                       # finalize the merge
```

VS Code highlights conflicts and offers one-click "Accept Current / Accept Incoming / Accept Both" buttons — much faster than hand-editing markers.

## 11. The GitHub side: PRs, reviews, and `gh` CLI

You already installed `gh` in chapter 02 and authenticated. Useful `gh` commands:

```bash
gh repo view                         # info about the current repo
gh repo clone diogo-arista/avd-101   # clone via gh (uses your gh auth)
gh pr create --fill                  # open a PR with auto-filled title/body
gh pr list                           # list open PRs
gh pr checkout 42                    # check out PR #42 locally for review
gh pr view 42 --web                  # open PR in browser
gh pr merge 42 --squash --delete-branch
gh issue create --title "..." --body "..."
gh workflow run my-workflow.yml      # trigger a GitHub Actions workflow
```

For a typical PR flow on this repo:

```bash
# 1. Sync local main with remote
git switch main
git pull

# 2. Branch
git switch -c chapter/03-feedback

# 3. Edit files, then:
git add 03-linux-bash-essentials.md
git commit -m "Tweak ch.03 §6 grep examples per review"
git push -u origin chapter/03-feedback

# 4. Open the PR
gh pr create --fill

# 5. After approval + merge, clean up
git switch main
git pull
git branch -d chapter/03-feedback
```

## 12. `.gitignore` and what *not* to commit

Your repo already has a `.gitignore` from chapter 02. The rules:

- **Never commit secrets.** Credentials, API tokens, SSH keys, `vault.yml` with cleartext, `.env`. If you do by mistake, *change the secret* immediately — removing the file from history is hard and the leak may already be indexed.
- **Don't commit generated artifacts** unless your workflow specifically requires it (AVD does — it commits `intended/configs/*.cfg` so PR reviews show config diffs).
- **Don't commit large binaries** to a normal Git repo. Use Git LFS for binaries >50 MB.
- **Don't commit user-specific configs** — VS Code `settings.json` with personal paths, editor backup files.

To check what's tracked:

```bash
git ls-files                # all tracked files
git ls-files --others --ignored --exclude-standard  # files being ignored
```

## 13. Tags and releases

When you want to mark a specific commit as "v1.0" or "production-deploy-2026-05-20":

```bash
git tag v1.0                         # lightweight tag
git tag -a v1.0 -m "First release"   # annotated tag (recommended)
git push --tags                      # push tags to remote
```

GitHub Releases let you attach a changelog + binaries to a tag — useful for AVD project repos to mark "this commit was deployed to prod."

## 14. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Repo** | A directory tracked by Git (has a `.git/` subfolder). |
| **Commit** | A snapshot with a message, author, and a unique hash. |
| **HEAD** | The current commit pointer (where you'd commit if you ran `git commit` now). |
| **Branch** | A named, movable pointer to a commit. |
| **Remote** | A reference to another copy of the repo (usually GitHub). |
| **Origin** | The conventional name for the default remote. |
| **Upstream** | Either (a) the original repo a fork came from, or (b) the remote branch a local branch tracks. |
| **Fetch** | Download remote changes, don't modify working tree. |
| **Pull** | Fetch + merge. |
| **Push** | Send local commits to a remote. |
| **Fork** | A personal copy of someone else's repo on GitHub. |
| **PR / MR** | Pull Request (GitHub) or Merge Request (GitLab) — a proposed change. |
| **Rebase** | Replay your commits on top of another base. Powerful, can rewrite history. |
| **Merge** | Combine two histories with a merge commit. Preserves history exactly. |
| **Conflict** | When Git can't auto-merge two changes; needs your manual resolution. |
| **Stash** | A temporary stash of uncommitted changes you can pop later (`git stash`, `git stash pop`). |

---

## 🧪 Exercise

1. **Branch + PR your first edit:**
   ```bash
   cd /Users/$USER/projects/avd-study
   git switch -c chapter/04-test
   echo "" >> 04-git-github.md
   echo "<!-- I read chapter 04 -->" >> 04-git-github.md
   git diff                         # confirm the change
   git add 04-git-github.md
   git commit -m "ch.04: add marker comment after first read"
   git push -u origin chapter/04-test
   gh pr create --fill
   ```
   Look at the PR in your browser. Then close (not merge) it — we don't need the comment in main.

2. **Reflog rescue:**
   ```bash
   git switch main
   echo "OOPS" > 00-network-automation-101.md
   git status
   git restore 00-network-automation-101.md     # recover from working tree
   git status                                    # clean again
   ```

3. **Conflict drill:**
   - On `main`, edit line 1 of README.md, commit.
   - Create a branch `bad-conflict`, edit line 1 of README.md *differently*, commit.
   - `git switch main`, `git merge bad-conflict`. Resolve the conflict, commit, then delete the branch.

4. **Read your own history:** `git log --oneline --graph` against the repo. Identify which commits I created from your conversation with me. Note the message style — emulate it for your own future commits.

5. **`gh` exploration:** `gh repo view --web` opens your repo in a browser. Then `gh pr list --state all` shows every PR ever. Find one of the canonical AVD examples PRs and read the diff: `gh pr view 4321 --repo aristanetworks/avd`.

---

**Next:** [05 — Devcontainers & VS Code](05-devcontainers-vscode.md)
