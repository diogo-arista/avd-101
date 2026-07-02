# Lab: Minimal AVD Project (Chapters 14–17)

A starter scaffold for your first AVD project, built up across the exercises in chapters 14–17.

## Structure

```
14-avd-min/
├── ansible.cfg
├── inventory.yml
├── group_vars/
│   ├── FABRIC.yml             # connection creds, management defaults
│   ├── DC1.yml                # design type, fabric name
│   ├── DC1_FABRIC.yml         # ASNs, IP pools, routing protocol
│   ├── DC1_SPINES.yml         # spine node definitions
│   ├── DC1_L3_LEAVES.yml      # leaf node definitions
│   ├── NETWORK_SERVICES.yml   # tenants / VRFs / SVIs
│   └── CONNECTED_ENDPOINTS.yml# servers and port mappings
├── build.yml
└── deploy.yml
```

## Prerequisites

- OrbStack VM with containerlab + cEOS running (chapter 02).
- Ansible + AVD collection installed (chapter 13 / repo root `requirements.yml`):
  ```bash
  ansible-galaxy collection install -r ../../requirements.yml
  ```

## Usage

```bash
cd labs/14-avd-min

# Build (local, safe — no device contact)
ansible-playbook build.yml

# Review generated configs
ls intended/configs/
git diff intended/

# Deploy (pushes to devices — confirm before running)
ansible-playbook deploy.yml
```

## Exercises

Each chapter's exercise section adds to this project:
- **Chapter 14**: populate `group_vars/` from scratch, run `build.yml`, inspect outputs.
- **Chapter 15**: run `deploy.yml` against the containerlab topology, then `validate.yml`.
- **Chapter 16**: read the generated artifacts in `intended/` and `documentation/`.
- **Chapter 17**: add ANTA tests and run `validate.yml` with custom catalogs.
