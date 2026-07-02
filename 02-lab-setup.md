# 02 — Lab Setup: containerlab + cEOS

> **Goal of this chapter:** end with a 4-node Arista fabric running locally, that you can `ssh` into and that AVD can target. We'll also install the tools (Ansible, AVD collection, Python) you'll use from chapter 03 onward.

You're on **macOS**. macOS is a friendly desktop but a slightly awkward host for containerlab, because containerlab fundamentally needs Linux networking primitives (network namespaces, veth pairs, Linux bridges). The good news: this is a solved problem, with a couple of standard patterns. We'll pick the most reliable one.

## 1. Choose a lab pattern

We're going to run a real Linux VM on your Mac and do all our work inside it. That gives us a clean Linux kernel for containerlab (which needs network namespaces, veth pairs, etc.) and matches how production CI pipelines run.

The two reasonable VM hosts on macOS in May 2026:

| Tool | Engine | Apple Silicon | Notes |
|---|---|---|---|
| **OrbStack** *(recommended)* | Apple Virtualization.framework | Native | Fast, free for personal use, stable on M-series Macs. We'll use this. |
| Multipass | QEMU (bundled) | Works but buggy on AS | Known regression in 1.16.2+ on Apple Silicon (`host-arm-cpu.sme` error). See appendix A if you want to try it anyway. |

> **Why a network engineer should care:** every CI/CD pipeline you'll touch eventually runs on Linux. Doing your lab work inside a Linux VM matches that reality and avoids "works locally but not in CI" surprises.

## 2. Prerequisites

You need:
- macOS (Apple Silicon or Intel — both work)
- ~15 GB free disk
- An [Arista.com account](https://www.arista.com/en/user-registration) (free) — required to download cEOS-lab images
- A terminal you're comfortable in (the built-in Terminal.app is fine; iTerm2 is nicer)

## 3. Install OrbStack and create the lab VM

### 3a. Install OrbStack

Install OrbStack via Homebrew (install Homebrew first from <https://brew.sh> if you don't have it):

```bash
brew install --cask orbstack
```

Open the OrbStack app once from `/Applications/OrbStack.app` (or via Spotlight). It'll ask for one-time permissions to set up its virtualization helper. Accept the defaults — you don't need Docker on the Mac side, just the Linux Machines feature.

> **What's OrbStack in one paragraph?** OrbStack is a macOS-native virtualization tool that uses Apple's `Virtualization.framework` to run Linux machines and Docker containers efficiently on Apple Silicon. Unlike Multipass (which wraps QEMU) or Docker Desktop (which runs a full Linux VM under the hood), OrbStack is designed from the ground up for Apple Silicon, which makes it faster, lighter, and free of the QEMU CPU-detection bugs we'd otherwise have to work around.

### 3b. Cap resource usage (optional but recommended)

OrbStack auto-grows resource usage based on what guests need, capped by global settings. If you have ≥16 GB of RAM, defaults are fine. To set explicit caps, open the OrbStack app → **Settings → System → Resources** and set:
- CPU limit: **4**
- Memory limit: **8 GB**

(These are *caps*, not *reservations* — OrbStack will use less when idle.)

### 3c. Create the Ubuntu machine

```bash
orb create ubuntu:jammy avdlab
```

`jammy` is Ubuntu 22.04 LTS — same as we planned for Multipass. This downloads the cloud image (~300 MB on Apple Silicon, ARM64) and provisions the machine. Takes ~30-60 seconds the first time, far quicker on subsequent rebuilds.

Verify:

```bash
orb list
```

You should see `avdlab` in `Running` state.

### 3d. Shell into the machine

```bash
orb -m avdlab
```

The `-m` flag selects the machine. Drop it and `orb` opens a shell into your default machine (whichever was created last).

You're now inside Ubuntu 22.04 on ARM64. Confirm:

```bash
uname -a            # should mention aarch64 GNU/Linux
cat /etc/os-release # should say Ubuntu 22.04
whoami              # should print your macOS username (OrbStack creates a matching user)
```

> **Convenient surprise:** OrbStack auto-creates a user with your macOS username and sets up passwordless sudo inside the machine. So you can `sudo apt install …` without prompts. You also get a shared file path: `/Users/<you>` on the Mac is visible inside the machine as the same path. We'll use this in chapter 03 to keep project files on the Mac while running tooling in Linux.

The rest of this chapter runs **inside the OrbStack machine** unless explicitly noted.

### 3e. Set up VS Code (strongly recommended)

You *can* do this whole curriculum from a plain terminal and `nano`/`vim` — **VS Code is not strictly mandatory**. But for the kind of work you're about to do (editing YAML, reading generated configs, browsing a project tree, hopping between Mac and the Linux machine, and especially driving containerlab), VS Code reduces friction by an order of magnitude. The rest of the chapter assumes you've set it up.

> **What's VS Code in one paragraph?** Visual Studio Code is a free, open-source code editor from Microsoft, ubiquitous in software and infrastructure work. It's a *text editor* at its core (like nano or vim) but with three things that make it especially useful here: (1) a file explorer + integrated terminal so you don't context-switch between windows, (2) **extensions** that add language support — YAML linting, Jinja2 highlighting, Ansible awareness, containerlab topology graphs, etc., and (3) **remote development modes** that let the editor run on your Mac while the files and tooling live somewhere else (an SSH host, an OrbStack machine, a devcontainer).

#### Why it matters for *this* lab specifically

| Capability | What it does for you |
|---|---|
| **Remote - SSH** extension | Opens a folder inside the OrbStack machine as if it were local. Edits, terminals, language servers, and other extensions all run *in Linux*; you see the result on your Mac. Eliminates the "is this file on the Mac or in the VM?" problem. |
| **Containerlab** extension (`srl-labs.vscode-containerlab`) | Auto-discovers your `.clab.yml` files; gives you a colour-coded sidebar of labs and nodes; right-click a lab to `deploy` / `destroy` / `redeploy` / `inspect`; right-click a node for shell / SSH / logs / packet capture; renders an **interactive topology graph** of your fabric. Once you have this, you'll rarely type `containerlab deploy` again. |
| **Dev Containers** extension | Lets a Git repo declare its dev environment (Python version, Ansible version, pre-installed AVD collection, etc.) in a `devcontainer.json`. Anyone who clones the repo and opens it in VS Code gets the identical environment in seconds. AVD ships such a devcontainer — we'll use it in chapter 05. |
| **YAML / Ansible / Jinja extensions** | Schema-aware completion and linting on AVD inputs, real-time error squiggles in playbooks, syntax highlighting in `.j2` templates. Saves hours of "why isn't this rendering?". |
| **Integrated terminal** | Run `containerlab deploy`, `ansible-playbook`, `git`, `ssh` — all without leaving the editor. |
| **Side-by-side diffs** | When AVD regenerates a `.cfg` file, the diff view shows you exactly what changed. Indispensable for code reviews. |

#### Crucial concept: where each extension actually runs

VS Code's remote-dev model splits extensions into two pots:

- **Local (Mac) extensions** — pure UI/connectivity, no language tooling. They establish and manage the connection to the remote target. Example: Remote-SSH, Dev Containers.
- **Workspace (remote) extensions** — anything that has to *read your files* or *invoke a tool*: YAML linters, Ansible, Jinja highlighters, **and the containerlab extension**. These have to run in the same OS as the files and binaries they operate on.

This means: after you connect VS Code to `avdlab@orb` via Remote-SSH, you **re-install the workspace extensions inside that remote target**. VS Code makes this a one-click operation — when you open the Extensions panel in a Remote-SSH window, each extension shows where it's installed and offers an "Install in SSH: avdlab@orb" button. Skip this and the containerlab/YAML/Ansible features simply won't activate.

#### Install VS Code on the Mac

```bash
brew install --cask visual-studio-code
```

Open VS Code once so it adds the `code` command to your `PATH` (or run **Shell Command: Install 'code' command in PATH** from the command palette — `Cmd+Shift+P`).

#### Install the local (Mac) extensions

These are the connectivity ones. From the Mac terminal:

```bash
code --install-extension ms-vscode-remote.remote-ssh
code --install-extension ms-vscode-remote.remote-containers
```

#### Connect VS Code to the OrbStack machine

OrbStack auto-registers an SSH host called `orb`; address a specific machine as `<machine>@orb`. Verify from the Mac terminal first:

```bash
ssh avdlab@orb            # should drop you into the avdlab machine, no password
```

If that works, in VS Code:

1. `Cmd+Shift+P` → **Remote-SSH: Connect to Host…**
2. Pick `avdlab@orb` (or type it if not auto-listed).
3. A new VS Code window opens; the **bottom-left corner shows `SSH: avdlab@orb`**, confirming you're remote.
4. **File → Open Folder…** → `/Users/<you>/projects/avd-study` (your project folder is shared into the OrbStack machine at the same path, so you'll see the curriculum repo immediately).

#### Install the workspace (remote) extensions

Now that VS Code is connected to the OrbStack machine, install the tooling extensions *into* that remote target. Easiest way is from the integrated terminal — `Ctrl+`` ` — which is already a Linux shell inside the VM:

```bash
code --install-extension srl-labs.vscode-containerlab
code --install-extension redhat.vscode-yaml
code --install-extension redhat.ansible
code --install-extension samuelcolvin.jinjahtml
```

Because you ran `code` from a remote terminal, it installs into the remote VS Code server, not your Mac. Confirm by opening the Extensions panel (`Cmd+Shift+X`): each of the four should show **"Installed in SSH: avdlab@orb"**, not "Local".

> **Why this is the magic moment:** from here on, every lab task — `containerlab deploy`, editing YAML, browsing the topology graph, dropping into a node's shell — runs *in the Linux machine* but you drive it from your Mac with one editor window. This is the workflow most network automation teams use day-to-day, and exactly the mental model you'll need for the devcontainer chapter (05) and the AVD chapters (14–16).

#### Using the containerlab extension (preview)

The first time you open a folder containing a `.clab.yml` file (we'll create one in §7), watch for the **Containerlab icon** in the VS Code Activity Bar (the vertical strip on the far left). Clicking it gives you:

- A **labs sidebar** showing all detected topologies — colour-coded green (deployed), red (failed), yellow (partial), grey (not deployed).
- **Right-click a topology** → Deploy, Destroy, Redeploy (with/without cleanup), Inspect, Save, Graph, Edit.
- **Right-click a node** under a deployed lab → Attach Shell, SSH, View Logs, Copy Properties (name/IP/kind), Packet Capture.
- A **graph view** that renders your topology as nodes and links — useful for sanity-checking a fabric you just wrote.

The first time you trigger a privileged action, the extension will prompt to add your user to the `clab_admins` group (which we already did in §5, so it'll be a no-op confirmation).

We'll come back to VS Code in **chapter 05 — Devcontainers & VS Code**, where the same editor plus the Dev Containers extension picks up the AVD devcontainer and gives you a full Ansible/AVD/Python environment with zero install. For now, the Remote-SSH + containerlab combo is what we'll use throughout the rest of this chapter.

## 4. Install Docker

containerlab needs Docker. Inside the VM:

```bash
# Install Docker using the official convenience script
curl -fsSL https://get.docker.com | sh

# Let your user run docker without sudo
sudo usermod -aG docker $USER

# Apply the group change without logging out
newgrp docker

# Verify
docker run hello-world
```

You should see the hello-world banner.

## 5. Install containerlab (sudo-less)

containerlab 0.63.0+ ships a **sudo-less mode** so you don't have to prefix every command with `sudo`. The official installer script sets this up for you: it installs the binary with the SUID bit set, creates a `clab_admins` Unix group, and adds your current user to it. Members of that group can run `containerlab deploy/destroy/exec` without `sudo`.

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

Apply the new group membership in the current shell (or just log out and back in):

```bash
newgrp clab_admins
```

Verify the SUID bit and your group membership:

```bash
ls -l $(which containerlab)   # expect -rwsr-xr-x (the 's' is the SUID bit)
groups                         # should include 'clab_admins' and 'docker'
containerlab version
```

> **What's SUID?** On Linux, a binary with the SUID bit set runs as its file owner (here, `root`) regardless of who invoked it. Combined with the `clab_admins` group restriction, this gives you root-level networking ops without putting `sudo` in your muscle memory.

⚠ **Security note straight from the containerlab docs:** anyone in `clab_admins` effectively has root on this host. On your personal lab VM that's fine; on a shared box, treat the group like the `docker` group — i.e., admin-only.

For the rest of this chapter, you can drop the `sudo` from every `containerlab` command. (If you ever see `permission denied`, you skipped `newgrp clab_admins` — log out and back in.)

## 6. Get cEOS-lab and load it into Docker

cEOS-lab is the containerized EOS image. It's free but gated behind an Arista.com login.

**On your Mac** (not in the VM):
1. Log in to <https://www.arista.com/en/support/software-download>.
2. Navigate to **EOS-CEOS** → pick a recent stable train.
3. ⚠ **Apple Silicon:** pick the file whose name contains **`-arm64`** (e.g., `cEOS64-lab-4.32.5.1M-arm64.tar.xz`). The `-x86_64` build will not run on an `arm64` host. Arista publishes both — you want the arm64 one.
4. Download the `.tar.xz` (~500-700 MB).

OrbStack shares your Mac home directory into the machine at the *same path*, so you don't need to transfer the file at all — just reference it directly from inside the machine:

```bash
# inside the OrbStack machine
ls ~/Downloads/cEOS64-lab-4.32.5.1M-arm64.tar.xz   # should "just work"
```

Load the image into Docker and tag it. The tag here is what containerlab and AVD will reference:

```bash
docker import ~/Downloads/cEOS64-lab-4.32.5.1M-arm64.tar.xz ceos:4.32.5.1M
docker images | grep ceos
# If `docker import` fails with a format error, try `docker load -i` instead —
# the correct command depends on how Arista packaged the specific release.
```

You should see `ceos    4.32.5.1M    <hash>    <size>`.

> **Why a network engineer should care:** the tag (`4.32.5.1M`) is how containerlab and AVD reference the image. You'll see this exact string in topology files. Multiple EOS versions can coexist as different tags — useful for testing upgrades. We use a short tag (`4.32.5.1M`) rather than the full filename (`cEOS64-lab-...-arm64`) so the topology file reads cleanly.

## 7. Build your first topology

A minimal 2-spine, 2-leaf topology is **already in the curriculum repo** at `labs/02-first-fabric/topology.clab.yml`. Because OrbStack shares `/Users/<you>` into the VM at the same path, you can `cd` directly to it inside the VM — no typing or copying needed:

```bash
cd /Users/$USER/projects/avd-study/labs/02-first-fabric
cat topology.clab.yml
```

(If you'd rather hand-type it once for the practice, the file contents are short — open it in VS Code and you can read it in 10 seconds.)

The file looks like this:

```yaml
name: avd-tour

topology:
  kinds:
    arista_ceos:
      image: ceos:4.32.5.1M

  nodes:
    spine1: { kind: arista_ceos }
    spine2: { kind: arista_ceos }
    leaf1:  { kind: arista_ceos }
    leaf2:  { kind: arista_ceos }

  links:
    - endpoints: [spine1:eth1, leaf1:eth1]
    - endpoints: [spine1:eth2, leaf2:eth1]
    - endpoints: [spine2:eth1, leaf1:eth2]
    - endpoints: [spine2:eth2, leaf2:eth2]
```

⚠ **Tag mismatch?** If your cEOS Docker tag (from §6) isn't `4.32.5.1M`, edit `topology.clab.yml` and change the `image:` line to match what `docker images | grep ceos` shows.

A few things to notice:
- **`kind`** tells containerlab what driver to use. `arista_ceos` knows how to bootstrap cEOS properly (sets the right boot env, handles the management interface, etc.).
- **`endpoints`** are veth pairs between two device interfaces. `eth1` on a cEOS container maps to `Ethernet1` inside EOS.
- **No management IPs specified.** containerlab puts every node on a Docker bridge (`clab` network) and assigns them static IPs from its management pool — typically `172.20.20.0/24`.

Bring it up — either from the terminal:

```bash
containerlab deploy -t topology.clab.yml
```

…or, if you set up the VS Code containerlab extension in §3e: open the **Containerlab** view in the Activity Bar, right-click `topology.clab.yml`, and select **Deploy**. Same result, less typing. (Either way, learn the CLI form too — CI pipelines and SSH-only sessions will need it.)

This takes ~2 minutes the first time (cEOS is slow to boot). On success you get a table:

```
+---+----------------+--------------+--------------+----------+---------+----------------+
| # |     Name       | Container ID |    Image     |   Kind   |  State  |  IPv4 Address  |
+---+----------------+--------------+--------------+----------+---------+----------------+
| 1 | clab-avd-tour- | abc123       | ceos:4.32... | arista_  | running | 172.20.20.2/24 |
|   | spine1         |              |              | ceos     |         |                |
...
```

## 8. Connect to a device

SSH (default credentials are `admin` / `admin`):

```bash
ssh admin@172.20.20.2
# accept the host key, then password: admin

leaf1>enable
leaf1#show version
```

You're talking to a real EOS instance. Try a few CLI commands you're familiar with — `show lldp neighbors` should already see the spine neighbors.

⚠ **Gotcha:** the host SSH key for each container changes every time you redeploy. Use `ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` for scripted access, or accept that you'll re-trust keys on rebuilds.

## 9. Enable eAPI

> **What's eAPI in one paragraph?** eAPI (EOS API) is Arista's built-in HTTP-based API. You POST a JSON payload containing a list of CLI commands; the switch runs them and POSTs back JSON containing the structured output. So instead of scraping `show interfaces` text, you get a real dict you can index into. It's the primary way AVD (and most Arista automation) talks to switches. We'll dissect it properly in chapter 10 — for now, you just need it enabled so AVD can reach the box.

Enable eAPI on every device. From the EOS CLI:

```
configure
management api http-commands
   protocol https
   no shutdown
   !
   vrf MGMT
      no shutdown
end
write
```

> In production you'd configure eAPI under the MGMT VRF. In the containerlab default network there is no MGMT VRF, so for now you can leave eAPI on the default VRF (omit the `vrf MGMT` block) — the lab's management network *is* the default VRF.

Confirm from the VM:

```bash
curl -k -u admin:admin -X POST \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"runCmds","params":{"version":1,"cmds":["show version"],"format":"json"},"id":1}' \
  https://172.20.20.2/command-api
```

You should get back a JSON blob containing the EOS version. **This is the protocol AVD uses to push and verify config.** Hold onto this curl command — we'll dissect it in chapter 10.

## 10. Install Python, Ansible, and the AVD collection

You'll need these for chapter 03 onward. Still inside the VM:

```bash
# Python 3 + pip + git
sudo apt update
sudo apt install -y python3 python3-pip python3-venv git
```

> **What's a Python virtual environment (`venv`)?** A self-contained folder with its own copy of Python and its own set of installed packages, isolated from the system Python. It exists so that project A's `ansible-core 2.16` doesn't collide with project B's `ansible-core 2.18`. You "enter" a venv by `source`-ing its activate script; you "leave" by typing `deactivate`. Without venvs, every `pip install` pollutes a single global namespace and breaks something eventually. We'll cover this properly in chapter 11.

```bash
# Create a virtual environment for this project
python3 -m venv ~/.venvs/avd
source ~/.venvs/avd/bin/activate

# Install Ansible and supporting libraries into the venv
pip install --upgrade pip
pip install "ansible-core>=2.16,<2.18" "pyavd[ansible]"
```

> **What's an Ansible collection?** A bundled package of Ansible roles, modules, and plugins published as a single unit. `arista.avd` is the collection that contains the AVD roles (`eos_designs`, `eos_cli_config_gen`, etc.). Collections are installed via `ansible-galaxy` (the Ansible-equivalent of `pip` or `npm`). We'll cover collections in detail in chapter 13.

```bash
# Install the AVD collection itself
ansible-galaxy collection install arista.avd
```

Verify:

```bash
ansible --version
ansible-galaxy collection list arista.avd
```

You should see Ansible 2.16+ and the AVD collection (latest 4.x or 5.x).

> ⚠ The `source ~/.venvs/avd/bin/activate` line "enters" the virtual environment. **You need to run this each new shell session.** A quick check: when active, your shell prompt is prefixed with `(avd)`. Add the `source` line to `~/.bashrc` once you're confident this is your default workflow.

## 11. Tear down and rebuild

You'll be rebuilding this lab a lot. Useful commands:

```bash
# Stop and remove the topology (keeps the topology.clab.yml file)
containerlab destroy -t labs/02-first-fabric/topology.clab.yml

# Stop and *remove the per-node state directory* (clab-avd-tour/)
containerlab destroy -t labs/02-first-fabric/topology.clab.yml --cleanup

# Bring it back
containerlab deploy -t labs/02-first-fabric/topology.clab.yml

# Restart a single node
docker restart clab-avd-tour-leaf1
```

⚠ **Gotcha:** `destroy` without `--cleanup` keeps the per-device state directory. That can be useful if you want config to persist across redeploys — or annoying if you want a fresh boot. Know which one you want.

## 12. Stopping the machine at the end of the day

To save battery / RAM, stop the machine when you're not working:

```bash
# from your Mac (anywhere — orb works from inside or outside)
orb stop avdlab

# resume later
orb start avdlab
orb -m avdlab          # opens a shell
```

You can also stop/start from the OrbStack menubar app if you prefer GUI.

Don't `orb delete avdlab` unless you're ready to start from scratch — that nukes the VM and everything inside it. (Project files stored under `~/...` on your Mac are safe; only files written *only* inside the VM are lost.)

## 13. Troubleshooting checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| `containerlab deploy` hangs at "starting node" for >5 min | Not enough RAM available to OrbStack | OrbStack app → Settings → System → Resources; raise memory cap |
| `docker: permission denied` | Forgot the `usermod -aG docker` step | Re-run it and `newgrp docker` (or close+reopen the orb shell) |
| `containerlab: command not found` after install | New `clab_admins` group not picked up | `newgrp clab_admins`, or close+reopen the orb shell |
| Can't `ssh admin@172.20.20.x` | cEOS still booting; takes 60-90s after `deploy` returns | Wait, then retry |
| `curl: (60) SSL certificate problem` | cEOS has a self-signed cert | Always use `-k` with curl against eAPI |
| `exec format error` when starting cEOS containers | You loaded the x86_64 cEOS build on an arm64 host | Re-download the `-arm64` cEOS image; re-`docker import` with the same tag |
| `orb` command not found on the Mac | OrbStack CLI shim not on PATH | Open the OrbStack app once; it installs the CLI on first launch |
| OrbStack machine sluggish, high disk usage | OrbStack data dir bloated by old layers | OrbStack app → Settings → System → Reset; or `docker system prune -a` inside the machine |
| AVD collection install fails on Python deps | Old pip/Python | Inside the venv: `pip install --upgrade pip setuptools wheel` |

## 14. What you just built

You have:
- A **Linux VM** (`avdlab`) that mirrors how CI runs.
- **Docker + containerlab** to spawn arbitrary topologies in seconds.
- A **running 4-node fabric** you can SSH into and call eAPI against.
- **Python + Ansible + the AVD collection** ready to drive that fabric.

From here on, every chapter's exercises will assume this environment is up. Get comfortable starting and stopping it.

---

## 🧪 Exercise

1. **Tear down and redeploy** the lab to make sure the workflow is muscle memory.
2. **SSH into both spines and both leaves** in parallel (open four terminal panes). Run `show lldp neighbors` on each. Confirm the link map matches your `topology.clab.yml`.
3. **Modify the topology**: add a third leaf (`leaf3`) with two uplinks (one to each spine). Redeploy. Confirm LLDP shows it. (You're learning containerlab as a side-effect — it's a network engineer's superpower.)
4. **Call eAPI** with the `curl` command from §9, but change `show version` to `show interfaces Ethernet1 status`. Read the JSON output — notice the *structure* of the response.
5. **In your notes**, write: "What's different about a containerlab fabric vs a real hardware fabric? What would I *not* learn from this lab?" (Hint: think transceivers, line cards, real LACP/MLAG hardware behavior, ARP tables at scale, control-plane policing under load.) Useful to be honest with yourself about lab limits.

---

**Next:** **03 — Linux & Bash Essentials for Automation** *(coming next batch)*

---

## Appendix A — Using Multipass instead of OrbStack

If you'd rather use Multipass (e.g., you're on an Intel Mac, or you prefer Canonical's tool), the rest of this chapter works identically — just swap the §3 steps for Multipass.

### Install + authenticate

```bash
brew install --cask multipass

# Multipass ≥1.13 requires daemon authentication on first use
sudo multipass set local.passphrase
multipass authenticate
```

### Launch the VM

```bash
multipass launch 22.04 \
  --name avdlab \
  --cpus 4 \
  --memory 8G \
  --disk 30G
```

### Shell into it

```bash
multipass shell avdlab
```

### Known issue on Apple Silicon (May 2026)

Multipass 1.16.2+ has an open bug (canonical/multipass#4851) where QEMU rejects the bundled CPU profile with:

```
qemu-system-aarch64: Property 'host-arm-cpu.sme' not found
```

If you hit this and the VM ends up in `Unknown` state, only two workarounds exist:

1. **Downgrade** to Multipass 1.16.1 (manual download from <https://github.com/canonical/multipass/releases>).
2. **Switch to OrbStack** — see §3.

For the curriculum we recommend (2). All later chapters assume an OrbStack machine but work the same inside any Linux VM with Docker, containerlab, Python, and Ansible installed.

### Multipass equivalents for OrbStack commands

| OrbStack | Multipass |
|---|---|
| `orb create ubuntu:jammy avdlab` | `multipass launch 22.04 --name avdlab --cpus 4 --memory 8G --disk 30G` |
| `orb -m avdlab` (shell) | `multipass shell avdlab` |
| `orb stop avdlab` | `multipass stop avdlab` |
| `orb start avdlab` | `multipass start avdlab` |
| `orb delete avdlab` | `multipass delete avdlab && multipass purge` |
| Shared `~/...` paths | `multipass mount ~/projects/avd-study avdlab:/home/ubuntu/avd-study` |
