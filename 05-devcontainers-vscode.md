# 05 — Devcontainers & VS Code

> **Goal of this chapter:** understand what a devcontainer is, why AVD ships one, and how to launch VS Code into a pre-built AVD environment so you can experiment without managing Python/Ansible versions by hand. You'll end with a working AVD devcontainer attached to your OrbStack VM.

## 1. Why devcontainers exist

Every "set up your dev environment" guide on the internet has the same shape: install Python X, pip-install A, B, C; install Ansible Y; install collections P, Q, R; oh and also set these three env vars and put this file at this path. After 45 minutes you have *almost* the right setup, and your version of pip-package-C is slightly newer than your colleague's, and now playbooks render differently.

**Devcontainers** are the fix. A devcontainer is:

- A **Docker container** that contains a project's entire dev environment — language runtime, OS packages, pip packages, ansible collections, linters, formatters, the works.
- A **`.devcontainer/devcontainer.json`** file inside the project repo that declares which image to use, which VS Code extensions to install, and what to run on first start.
- Combined with the **Dev Containers** VS Code extension: open the project, click "Reopen in Container", and VS Code spawns the container, attaches itself to it, and presents a normal editor experience that's actually running *inside* the container.

The result: clone the repo → open it → 30 seconds later you have the *exact* environment the project maintainers use, every time, identical on every machine.

> **Why this matters for network automation:** AVD pins specific versions of `ansible-core`, `pyavd`, `arista.eos`, the `arista.cvp` collection, several Python libraries, and more. The AVD project ships a devcontainer so that anyone — you, your team, your CI runner — gets the right combination every time.

## 2. Devcontainer vs virtual environment vs raw install

| Approach | Isolates | Reproducible? | Setup time |
|---|---|---|---|
| **Raw pip + ansible-galaxy install** (what chapter 02 §10 did) | Just Python packages | Sort of — you must remember which versions | Manual |
| **Python venv** | Python packages, isolated per project | Mostly — but OS deps still drift | A few minutes per project |
| **Devcontainer** | OS, Python, all packages, all tools, plus the editor's extensions | Yes — declared in a file in the repo | Seconds, after the image builds once |

You'll use all three at different times. Devcontainers shine when:
- Multiple people work on the same project.
- The project has many moving parts (AVD: Ansible, Python, Jinja, Ruby for docs, Node for some tooling).
- You want CI and local dev to be bit-identical.

## 3. The pieces

A devcontainer is just three things:

```
my-repo/
├── .devcontainer/
│   ├── devcontainer.json     # spec: which image, which extensions, post-create scripts
│   └── Dockerfile            # optional: custom image definition
└── (rest of the repo)
```

A minimal `devcontainer.json`:

```json
{
  "name": "AVD Study Lab",
  "image": "mcr.microsoft.com/devcontainers/python:3.11",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "redhat.ansible",
        "redhat.vscode-yaml",
        "samuelcolvin.jinjahtml"
      ]
    }
  },
  "postCreateCommand": "pip install 'ansible-core>=2.16,<2.18' 'pyavd[ansible]' && ansible-galaxy collection install arista.avd"
}
```

When you "Reopen in Container":
1. Docker pulls the `image` (or builds the Dockerfile).
2. VS Code starts the container with your repo mounted at `/workspaces/<repo>`.
3. VS Code's server is installed inside the container.
4. Listed `extensions` are installed into the container's VS Code server.
5. The `postCreateCommand` runs once (here: install Ansible and the AVD collection).
6. VS Code reopens, attached to the container — you're now "inside" it. Bottom-left status bar shows `Dev Container: AVD Study Lab`.

## 4. Where the container actually runs

Two layers of "remote" can stack here. Pay attention:

| Scenario | Container runs on | Files come from |
|---|---|---|
| **Mac with Docker Desktop** | Mac | Mac local FS |
| **OrbStack with Docker** | OrbStack VM (Mac is just UI) | Mac (shared) or VM-local |
| **VS Code Remote-SSH into VM, then "Reopen in Container"** | Inside the VM | The repo on the VM |

For us, scenario 3 is the usual pattern: VS Code on Mac → SSH to OrbStack VM → "Reopen in Container" launches a container *inside the VM* using the VM's Docker daemon. That keeps everything Linux-native and matches what CI runners do.

## 5. The official AVD devcontainer

The Arista AVD project ships an official devcontainer image: `ghcr.io/aristanetworks/avd:latest` (and version-pinned tags like `ghcr.io/aristanetworks/avd:5.2.0`). It includes:

- Python with `ansible-core`, `pyavd`, all required Python deps.
- The `arista.avd`, `arista.eos`, `arista.cvp` Ansible collections.
- Linting tools: `ansible-lint`, `yamllint`.
- Documentation tooling: `mkdocs` and the AVD documentation theme.
- Pre-configured VS Code extensions and settings for AVD work.

The recommended way to use it is to drop a `.devcontainer/devcontainer.json` into any AVD project that references the image. Then anyone opening that project gets the same toolchain.

## 6. Adding a devcontainer to *this* curriculum repo

Let's actually do it. From your Mac (or in the VS Code Remote-SSH terminal — same shared path), in the repo root:

```bash
mkdir -p .devcontainer
```

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "AVD-101 Study",
  "image": "ghcr.io/aristanetworks/avd:latest",
  "customizations": {
    "vscode": {
      "extensions": [
        "redhat.ansible",
        "redhat.vscode-yaml",
        "samuelcolvin.jinjahtml",
        "srl-labs.vscode-containerlab",
        "ms-python.python"
      ],
      "settings": {
        "files.associations": {
          "*.j2": "jinja"
        }
      }
    }
  },
  "remoteUser": "avd",
  "workspaceFolder": "/workspaces/avd-101",
  "postCreateCommand": "ansible --version && ansible-galaxy collection list arista.avd"
}
```

What each key does:

- **`image`** — the AVD-maintained container image. Pulled from GitHub Container Registry.
- **`customizations.vscode.extensions`** — installed into the container's VS Code server on first open. Note: these are the *workspace* extensions (the ones that touch your code), not Mac-side connectivity extensions.
- **`settings`** — VS Code settings scoped to this container. Here we tell VS Code that `*.j2` files are Jinja, so the highlighter activates.
- **`remoteUser`** — which user inside the container VS Code runs as. The AVD image creates a non-root `avd` user.
- **`postCreateCommand`** — runs once after the container is created. Here it's a sanity check that prints Ansible and collection versions.

## 7. Opening it

With the file in place, in VS Code:

1. `Cmd+Shift+P` → **Dev Containers: Reopen in Container** (or click the green icon in the bottom-left, then "Reopen in Container").
2. First time, you'll see "Starting Dev Container…" then a build/pull progress log. ~2-5 min the first time depending on bandwidth.
3. When it finishes, the bottom-left badge changes to **Dev Container: AVD-101 Study**.
4. The integrated terminal is now a shell *inside the container*. Try:
   ```bash
   whoami            # avd
   pwd               # /workspaces/avd-101
   ansible --version
   ansible-galaxy collection list arista.avd
   ```

You now have the entire AVD toolchain ready, with no local install pollution.

## 8. Working *across* containers: containerlab + devcontainer

Important distinction:

- The **devcontainer** is where you run *Ansible* / *Python* / *AVD* (the tools).
- The **containerlab** topology is where your *cEOS switches* run.

These are two separate Docker setups. In our OrbStack VM, both share the same Docker daemon. So inside the AVD devcontainer, you can:

```bash
# from inside the devcontainer
docker ps                # see the containerlab cEOS nodes
ping leaf1               # if you set up /etc/hosts or use clab's DNS
ansible all -m ping      # if your inventory references the lab nodes' IPs
```

The trick is that the devcontainer needs Docker access to "see" containerlab nodes. The official AVD devcontainer supports this via Docker socket mounting (`/var/run/docker.sock`), but only if you start it with the right mounts. To enable it, add to `devcontainer.json`:

```json
"mounts": [
  "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"
]
```

⚠ Mounting the Docker socket gives container code full control over Docker. Fine on a personal lab; never do it on shared/untrusted infrastructure.

## 9. Rebuilding when the spec changes

If you edit `devcontainer.json` (e.g., add an extension or change the image tag), you must rebuild the container:

- `Cmd+Shift+P` → **Dev Containers: Rebuild Container** — keeps the same name, pulls/rebuilds the image.
- **Dev Containers: Rebuild Without Cache** — when something is stuck and you want a clean slate.

The first build is slow; rebuilds are usually fast because Docker layers are cached.

## 10. Pinning versions

`image: ghcr.io/aristanetworks/avd:latest` floats — your container will drift as Arista publishes new images. For a learning repo that's fine. For a production AVD project, pin:

```json
"image": "ghcr.io/aristanetworks/avd:5.2.0"
```

This way every contributor and every CI run uses the exact same toolchain version. Bump the tag intentionally, in a PR, with the AVD version's release notes in hand.

## 11. When NOT to use a devcontainer

- **Quick exploration** — running a one-off `ansible-playbook` against a lab. Just use the venv from chapter 02.
- **Tooling that needs the host kernel directly** — containerlab itself needs the Linux kernel of the VM. You can't run containerlab *inside* the devcontainer (well, you can with Docker-in-Docker, but it's painful).
- **You're already 100% inside the orb VM with the right venv active** — adding a container layer is overkill.

The right mental model: the orb VM is your *workstation*; the devcontainer is your *AVD project's frozen workbench*; the containerlab nodes are your *DUTs*. Each does one thing well.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Devcontainer** | A Docker container declared in a project repo as the canonical dev environment. |
| **`devcontainer.json`** | The spec file (lives in `.devcontainer/`). |
| **Workspace folder** | Where your repo files appear inside the container (default `/workspaces/<repo>`). |
| **Features** | Reusable add-ons published by the devcontainers community (e.g., install Docker CLI, install Node). |
| **Post-create command** | One-time script run after container creation. |
| **VS Code Server** | The headless half of VS Code that runs in the remote (container/SSH host). The "client" stays on your Mac. |
| **GHCR** | GitHub Container Registry — where the AVD image is hosted. |

---

## 🧪 Exercise

1. **Add the devcontainer to your repo** as in §6, commit, push:
   ```bash
   git switch -c chapter/05-devcontainer
   mkdir -p .devcontainer
   # (create devcontainer.json with the contents from §6)
   git add .devcontainer/devcontainer.json
   git commit -m "Add AVD devcontainer for the curriculum repo"
   git push -u origin chapter/05-devcontainer
   gh pr create --fill
   ```
2. **Open it**: `Cmd+Shift+P` → **Dev Containers: Reopen in Container**. Wait for first build (grab a coffee).
3. **Verify the environment** inside the container:
   ```bash
   ansible --version
   ansible-galaxy collection list arista.avd
   python --version
   ```
   All three should produce sensible output without you having installed anything.
4. **Test the YAML schema awareness**: open `labs/02-first-fabric/topology.clab.yml`. Hover keys — you should see hover docs. Type a deliberately wrong key (`nodez:` instead of `nodes:`) and watch for the red squiggle.
5. **Switch back out**: bottom-left badge → "Reopen Folder in SSH" to go back to the plain VM environment. Confirm `ansible --version` is *different* now (whichever version your venv has, vs the devcontainer's pinned one).
6. **Merge the PR** and `git pull` on `main`. From now on, anyone (including future-you on a new machine) can clone this repo and have the AVD toolchain in 2 minutes.

---

**Next:** [06 — YAML Deep Dive](06-yaml-deep-dive.md)
