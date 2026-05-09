# SimpleX SMP Server — Raspberry Pi 4

Deployment, troubleshooting and validation guides for running a self‑hosted SimpleX SMP server on a Raspberry Pi 4, using Docker and a minimal, reproducible setup.

## what this is

This repository documents a full workflow to:

- prepare a Raspberry Pi 4 as a dedicated SimpleX server host  
- configure networking (DuckDNS, port forwarding, firewall, CGNAT decisions)  
- deploy SimpleX SMP via Docker with a sane TLS model  
- validate the deployment step by step and keep a record of what was done  

It is written as a practical, opinionated runbook — not as generic documentation.

## documents

All guides live under `docs/`:

- `simplex-selfhosted-rpi4-guide.md`  
  Long‑form deployment guide (concepts + detailed steps).

- `01-deployment-checklist.md`  
  Linear checklist to follow during deployment.  
  Meant to be filled in as you go (values, decisions, screenshots).

- `02-troubleshooting-quickref.md`  
  Quick reference for common failures (networking, Docker, TLS, client connection).

- `03-validation-report-template.md`  
  Template to record what was deployed: hardware fingerprint, OS version, network plan, final server address and tests performed.

## scope and assumptions

- Hardware: Raspberry Pi 4 (2 GB RAM minimum, 4 GB+ recommended).  
- OS: recent Raspberry Pi OS (Bookworm or Bullseye), with SSH access.  
- Network: you can log into your router, understand basic port forwarding, and you know whether you are behind CGNAT.  
- Skills: comfortable with SSH, basic shell usage, editing text files, and copy‑pasting commands correctly.

The guides are designed for reproducibility and auditability: each critical decision is written down, and each command has a place where its output can be recorded.

## relation to other projects

This repository is part of a broader off‑grid communication stack:

- `rawmesh-node` — Reticulum / NomadNet / LXMF node on Raspberry Pi 4  
- `mesh-guides` — Meshtastic EU868 documentation (PT/EN/ES)  
- `tdeck-guides` — T‑Deck / T‑Deck Plus Meshtastic guides  
- `cyphertools` — applied cryptographic tools and key management notes

## license

This project is distributed under the terms described in the `LICENSE` file in this repository.
