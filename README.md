# Infrastructure Journal

A working record of building and running Linux servers — what I built, what
broke, and how I diagnosed it.

Written as I go, not reconstructed afterwards.

---

## Current stack

| | |
|---|---|
| **Host** | Vultr Cloud Compute · Shared CPU · 1 GB RAM · Johannesburg |
| **OS** | Ubuntu 26.04 LTS |
| **Access** | SSH, key-only. Password authentication disabled, root SSH login disabled. |
| **Firewall** | `ufw`, deny-by-default. Ports 22, 80, 443 open. |
| **Updates** | `unattended-upgrades` — security patches applied automatically |
| **Memory** | 2.3 GB swap, swappiness tuned to 10 |
| **Containers** | Docker CE from the official repository |
| **Running** | n8n, bound to `127.0.0.1` only, state in a named volume |

---

## How this repository is organised

**`builds/`** — what I constructed, in what order, and why each decision was
made. The reasoning matters more than the commands.

**`incidents/`** — things that broke. Symptom, what I checked and in what order,
what it turned out to be, how I fixed it, and what I would check first next time.

**`stack/`** — configuration files, versioned so the whole setup can be rebuilt
from scratch.

---

## Progress

**Phase 0 — Server setup and hardening** ✅
Provisioning, non-root user, SSH key authentication, SSH hardening, firewall,
swap tuning, automatic security updates. Verified across a reboot.

**Phase 1 — Docker** *(in progress)*
Installed from the official repository with signature verification. n8n running
in a container, bound to localhost, reached over an SSH tunnel. Application state
persisted in a Docker volume — verified by destroying the container twice, with
and without a volume attached.

**Phase 2 — Domain, reverse proxy, HTTPS** *(planned)*

---

## Working principles

**Never cut the branch you are sitting on.** When changing anything that
controls your own access: know the escape route first, keep a working session
open, test before applying, verify from a fresh connection before letting go.

**Check state before changing state.** Confirm something does not already exist
before building it.

**A valid config is not a correct config.** Syntax checks verify grammar, not
intent. Confirm the setting actually took effect.

**Applying and persisting are separate jobs.** Ask of every change: will this
still be true after a reboot? Then restart and check.

**When unsure, ask the machine.** Logs and status commands already know what
happened. Reading them beats guessing.

**Bind services to `127.0.0.1` unless they genuinely need public reach.** Docker
publishes ports below the host firewall, so `ufw` will not save you.

**Configuration belongs in Git. Data belongs in a volume.**

**Prove behaviour rather than assuming it.** Destroying something deliberately
teaches the boundary more reliably than reading about it.
