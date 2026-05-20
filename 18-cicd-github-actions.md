# 18 — CI/CD Basics with GitHub Actions

> **Goal of this chapter:** set up a real GitHub Actions workflow that lints, builds, and (optionally) validates your AVD project on every pull request. By the end you should be able to read any GitHub Actions YAML, write a basic pipeline, and understand the patterns the AVD community uses.

This is the chapter that turns AVD from "a thing I run locally" into "a thing my team runs through code review."

## 1. Why CI/CD for network configs

The full case from [chapter 01 §5](01-avd-guided-tour.md):

- **CI = "did we break anything?"** — every PR runs your linters, builds, and (where safe) validates. Bad changes can't merge.
- **CD = "ship it"** — when CI passes on `main`, automation deploys the change.

For an AVD repo the pipeline typically:

1. On PR open/push:
   - Lint YAML and Ansible.
   - Run `build.yml` to confirm configs render without error.
   - Diff `intended/configs/*.cfg` between the PR and `main`; post the diff as a PR comment.
   - (Optional) Stand up a containerlab fabric, deploy AVD to it, run `eos_validate_state`. Post the report.
2. On merge to `main`:
   - Re-run `build.yml`.
   - Run `deploy.yml --check --diff` against staging. If clean, deploy.
   - Run `validate.yml` after deploy. If failed, alert + auto-rollback.

You can scale from "just lint and build on PR" up to the full thing. Start small.

## 2. GitHub Actions in one screen

GitHub Actions is GitHub's built-in CI service. The unit is a **workflow** — a YAML file under `.github/workflows/`. Each workflow contains **jobs**; each job runs on a **runner** (a fresh VM or container) and consists of **steps** (shell commands or pre-built **actions**).

Minimal workflow:

```yaml
# .github/workflows/hello.yml
name: hello
on: [push, pull_request]

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Say hi
        run: echo "Hello from GitHub Actions"
      - name: Check out the repo
        uses: actions/checkout@v4
      - name: List files
        run: ls -la
```

Push this file, open the repo on GitHub → **Actions** tab → watch it run.

Key concepts:

- **`on:`** — what triggers the workflow (`push`, `pull_request`, `schedule`, `workflow_dispatch` for manual, etc.).
- **`jobs:`** — one or many; can depend on each other (`needs:`).
- **`runs-on:`** — `ubuntu-latest`, `macos-latest`, `windows-latest`, or a self-hosted runner.
- **`steps:`** — sequential commands.
- **`uses:`** — invokes a published action (from GitHub Marketplace or a public repo).
- **`run:`** — runs a shell command.

## 3. Linting AVD inputs in CI

A useful first workflow: run `yamllint` and `ansible-lint` on every PR.

```yaml
# .github/workflows/lint.yml
name: lint
on:
  pull_request:
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip

      - name: Install linters
        run: pip install yamllint ansible-lint

      - name: Run yamllint
        run: yamllint .

      - name: Run ansible-lint
        run: ansible-lint
```

After committing this, every PR shows a green ✅ or red ❌ next to the linter step. Reviewers see status at a glance.

## 4. Building AVD configs in CI

Step 1: render the configs and confirm no error.

```yaml
# .github/workflows/build.yml
name: build
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install AVD
        run: |
          pip install "ansible-core>=2.16,<2.18" "pyavd[ansible]"
          ansible-galaxy collection install arista.avd

      - name: Build
        working-directory: labs/14-avd-min
        run: ansible-playbook -i inventory.yml build.yml

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: intended-configs
          path: labs/14-avd-min/intended/
```

Now every PR run produces downloadable `intended/configs/` you can inspect.

## 5. Posting config diffs as PR comments

The real "wow" step: when a PR changes inputs, comment with the resulting `.cfg` diff.

Outline:

1. Check out the PR branch.
2. Run build. Save `intended/configs/` as `pr-configs/`.
3. Check out `main`. Run build. Save as `main-configs/`.
4. Diff `main-configs` vs `pr-configs`.
5. Post the diff to the PR via `gh pr comment`.

```yaml
# .github/workflows/config-diff.yml
name: config diff
on:
  pull_request:
    paths:
      - "labs/14-avd-min/**"

jobs:
  diff:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - name: Check out PR
        uses: actions/checkout@v4
        with:
          path: pr
      - name: Check out main
        uses: actions/checkout@v4
        with:
          ref: main
          path: base

      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }

      - name: Install
        run: |
          pip install "ansible-core>=2.16,<2.18" "pyavd[ansible]"
          ansible-galaxy collection install arista.avd

      - name: Build PR
        run: ansible-playbook -i pr/labs/14-avd-min/inventory.yml pr/labs/14-avd-min/build.yml

      - name: Build base
        run: ansible-playbook -i base/labs/14-avd-min/inventory.yml base/labs/14-avd-min/build.yml

      - name: Compute diff
        id: diff
        run: |
          diff -u base/labs/14-avd-min/intended/configs/ pr/labs/14-avd-min/intended/configs/ > /tmp/diff.txt || true
          {
            echo 'diff<<EOF'
            head -c 60000 /tmp/diff.txt
            echo
            echo 'EOF'
          } >> "$GITHUB_OUTPUT"

      - name: Post diff comment
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          echo "${{ steps.diff.outputs.diff }}" | gh pr comment "$PR_URL" --body-file -
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
```

This is a simplified version of what production AVD repos do. Often this comment is auto-updated on every push.

## 6. Validating in a containerlab fabric inside CI

GitHub-hosted runners can run Docker. So you can spin up containerlab + cEOS inside a CI job, deploy AVD, validate, tear down.

Caveats:
- The cEOS-lab image is gated — you need to host it privately (e.g., in your repo's container registry) and authenticate.
- Each spin-up adds 2-5 minutes.
- Memory is constrained — keep topologies small (≤6 nodes).

Outline (sketch):

```yaml
- name: Pull cEOS image
  run: docker pull ghcr.io/<your-org>/ceos:4.32.5.1M
- name: Install containerlab
  run: bash -c "$(curl -sL https://get.containerlab.dev)"
- name: Deploy topology
  run: containerlab deploy -t labs/02-first-fabric/topology.clab.yml
- name: Wait for boot
  run: sleep 90
- name: Enable eAPI on all nodes
  run: # scripted curl, or ansible playbook
- name: AVD deploy
  run: ansible-playbook deploy.yml
- name: AVD validate
  run: ansible-playbook validate.yml
- name: Upload reports
  uses: actions/upload-artifact@v4
  with:
    name: validation-reports
    path: labs/14-avd-min/reports/
- name: Tear down
  if: always()
  run: containerlab destroy -t labs/02-first-fabric/topology.clab.yml --cleanup
```

This is the gold standard: a PR can fail because it would break the fabric, *before* anyone deploys it for real.

## 7. Secrets in CI

CI needs credentials for SSH/eAPI/CloudVision. Never hardcode. Use **GitHub Secrets**:

1. Repo → Settings → Secrets and variables → Actions → New repository secret.
2. Reference in workflow via `${{ secrets.NAME }}`.

```yaml
- run: ansible-playbook deploy.yml
  env:
    ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASS }}
    EOS_USERNAME: ${{ secrets.EOS_USER }}
    EOS_PASSWORD: ${{ secrets.EOS_PASS }}
```

For Ansible Vault, store the vault password as a secret, write it to a file at job start:

```yaml
- run: echo "$VAULT_PASS" > /tmp/.vault_pass
  env: { VAULT_PASS: ${{ secrets.VAULT_PASS }} }
- run: ansible-playbook deploy.yml --vault-password-file /tmp/.vault_pass
```

⚠ **GitHub never logs secret values.** But anyone who can write workflow files can exfiltrate them. Lock branch protection: require PRs into protected branches, require reviews from CODEOWNERS for `.github/workflows/`.

## 8. Concurrency, matrices, and caches

### Parallel builds across versions

```yaml
strategy:
  matrix:
    avd_version: ["5.1.0", "5.2.0"]
steps:
  - run: ansible-galaxy collection install arista.avd:==${{ matrix.avd_version }}
  - run: ansible-playbook build.yml
```

### Cancel old runs on new pushes

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Cache pip downloads

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.11"
    cache: pip
```

These are small but compound — fast CI is good CI.

## 9. CD: deploying from CI to real fabric

The simplest CD pattern: run `deploy.yml` on merge to `main`, only if `build.yml` passed.

```yaml
on:
  push:
    branches: [main]

jobs:
  build:
    # …
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production       # GitHub Environment protection (gating)
    steps:
      - uses: actions/checkout@v4
      - # … install AVD …
      - run: ansible-playbook deploy.yml --check --diff > /tmp/diff.txt
      - name: Upload diff for record
        uses: actions/upload-artifact@v4
        with: { name: deploy-diff, path: /tmp/diff.txt }
      - run: ansible-playbook deploy.yml
      - run: ansible-playbook validate.yml
```

GitHub Environments add a manual approval step (`environment: production` with required reviewers) — common for production fabrics. The CI pipeline does all the work; a human just clicks "deploy."

## 10. Self-hosted vs GitHub-hosted runners

| | GitHub-hosted | Self-hosted |
|---|---|---|
| Setup | None — just `runs-on: ubuntu-latest` | Install runner agent on your infrastructure |
| Cost | Free for public repos; quota for private | Your infrastructure cost |
| Access to internal network | None | Yes — runner is on your network |
| Reach into production fabrics | Hard (firewall) | Easy |

For network CD against private fabrics, self-hosted runners are nearly always the right answer.

## 11. The AVD community's CI patterns

Browse <https://github.com/aristanetworks/avd/tree/devel/.github/workflows> for examples. You'll see:

- Matrix builds across Ansible and AVD versions.
- Cached collection installs.
- Auto-publishing of Markdown docs to a GitHub Pages site.
- Spin-up of a containerlab fabric for integration tests.

These workflows are open-source — copy and adapt.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **CI** | Continuous Integration — run tests on every change. |
| **CD** | Continuous Deployment — auto-deliver passing changes. |
| **Workflow** | A YAML file under `.github/workflows/`. |
| **Job** | A unit that runs on a single runner. |
| **Step** | A command or action within a job. |
| **Action** | A reusable component (`actions/checkout`, etc.). |
| **Runner** | The VM or container where a job executes. |
| **Self-hosted runner** | A runner you operate, on your infra. |
| **Matrix** | Run the same job across multiple parameter combos. |
| **Secret** | Encrypted value injected as env var. |
| **Environment** | A GitHub feature for grouping secrets + adding approval gates. |
| **`gh`** | GitHub CLI — automate GitHub operations from scripts. |

---

## 🧪 Exercise

1. **First workflow**: create `.github/workflows/lint.yml` per §3. Commit, push, watch it run.

2. **Build workflow**: add `.github/workflows/build.yml` per §4. Confirm green on PR.

3. **Break it on purpose**: introduce a YAML typo in `labs/14-avd-min/group_vars/DC1.yml`. Push to a branch, open PR. CI should fail — read the logs to find the line you broke. Fix, push again, watch CI go green.

4. **Diff comment**: implement §5 (simplified — just diff and `echo`, skip the PR comment for now). Confirm the diff text is meaningful when you change a single VLAN.

5. **Read a real AVD workflow**: browse <https://github.com/aristanetworks/avd/blob/devel/.github/workflows/test_pyavd.yml> (or similar). Identify the parts: triggers, jobs, matrix, caching.

6. **(Stretch) Containerlab in CI**: try §6 with your own fork. You'll need to push the cEOS image to GHCR with auth and reference it in your workflow. This is significantly fiddlier but it's how real teams run network CI.

---

## Wrapping up the curriculum

You now have:

- A working OrbStack + containerlab + cEOS lab on your Mac.
- A VS Code Remote-SSH workflow tuned for AVD.
- A live curriculum repo on GitHub with everything tracked and versioned.
- Reading-level fluency in YAML, JSON, Jinja2, Python, Ansible, Git, and Linux/Bash.
- A mental model for APIs (REST, eAPI, gNMI, NETCONF) and data models (OpenConfig, YANG).
- A full AVD project: inputs → build → deploy → validate.
- CI that catches problems before they hit production.

What now? Pick a real change at your day job — add a VLAN, decom a leaf, change BGP timers — and try to express it as an AVD YAML change. The first one is the hardest. The hundredth is muscle memory.

When something breaks, **read the artifacts**: the intended config, the structured config, the validate report. They tell you almost everything.

Welcome to network automation.
