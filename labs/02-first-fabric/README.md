# Lab: First Fabric (Chapter 02)

A minimal 2-spine, 2-leaf containerlab topology used in [chapter 02 §7](../../02-lab-setup.md) to validate your local lab environment.

## How to deploy

Inside the OrbStack `avdlab` machine (the path is shared with your Mac, so this works whether you `cd` from the Mac or from inside the VM):

```bash
cd /Users/<you>/projects/avd-study/labs/02-first-fabric
containerlab deploy -t topology.clab.yml
```

Or, in VS Code Remote-SSH connected to `avdlab@orb`, right-click `topology.clab.yml` in the Containerlab sidebar → **Deploy**.

## Prerequisites

- OrbStack VM running with Docker + containerlab installed (chapter 02 §3–5).
- `ceos:4.32.5.1M` image present in the VM's Docker (chapter 02 §6). If your cEOS tag differs, edit `topology.clab.yml` and change `image: ceos:4.32.5.1M` accordingly.

## Cleanup

```bash
containerlab destroy -t topology.clab.yml --cleanup
```
