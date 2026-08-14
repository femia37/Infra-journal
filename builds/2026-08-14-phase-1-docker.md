# Phase 1 — Docker: installation, containers, and persistent storage

**Date:** 2026-08-14
**Machine:** Vultr Cloud Compute · Ubuntu 26.04 LTS · 1 GB RAM
**Outcome:** n8n running in a container, data persisting across container destruction

---

## Goal

Install Docker from source, run a real application in a container, and establish
where its data lives — deliberately proving what survives container deletion and
what does not.

The application is n8n, a workflow automation tool. It is the thing this server
exists to host.

---

## 1. Installed Docker from the official repository

Deliberately did **not** use `apt install docker.io`. Ubuntu's repositories
prioritise stability and freeze package versions, which is correct for the base
system and unhelpful for fast-moving tools — the packaged version can be a year
behind. It is also a differently-packaged build from the one most documentation
assumes.

Added Docker's own repository instead, which requires three steps:

```bash
# 1. Install the vendor's signing key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
     -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 2. Add the repository, referencing that key
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Install
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io \
     docker-buildx-plugin docker-compose-plugin -y
```

**Why signing matters.** Adding a third-party repository means instructing the
system to install software from a new source, with root privileges. Cryptographic
signing is what makes that safe: the vendor signs every package with a private
key, the system verifies against the installed public key, and refuses anything
whose signature does not match. It proves both origin and that nothing was
tampered with in transit.

**The pattern is identical for every third-party repository** — key, source file,
install. Only the URLs change.

**Always take these commands from the vendor's own documentation**, never from a
blog or forum answer. A stale URL may now point somewhere else entirely.

Verified that the service was enabled at boot, not merely running:

```bash
systemctl is-enabled docker      # enabled
systemctl status docker          # active (running)
```

## 2. Removed the need for `sudo` — and understood the trade-off

```bash
sudo usermod -aG docker $USER
```

**Why `sudo` was needed at all.** Docker is two programs: a daemon running as
root that does the actual work, and a client that sends it instructions. They
communicate through a socket file at `/var/run/docker.sock`, owned by root with
group `docker`. Without membership of that group, the client cannot open the
socket, and the request never reaches the daemon.

**The security implication, stated plainly.** Membership of the `docker` group is
equivalent to root access. The daemon runs as root and executes whatever it is
asked — including starting a container with the host's entire filesystem mounted.
From inside that container, as root, any file on the host can be modified. No
password prompt is involved at any point.

On a single-admin machine this changes nothing, since the user already has sudo.
**On a shared machine it is a decision that should be made knowingly.**

## 3. Ran n8n, bound to localhost only

```bash
docker run -d \
  --name n8n \
  -p 127.0.0.1:5678:5678 \
  -e GENERIC_TIMEZONE="Africa/Lagos" \
  -e TZ="Africa/Lagos" \
  -e N8N_SECURE_COOKIE=false \
  n8nio/n8n
```

**The `127.0.0.1` prefix is the important part.** Without it, Docker publishes
the port on every address the machine has, including the public one.

**n8n on first run has no authentication.** Whoever reaches it first creates the
owner account, and thereby gains every credential subsequently connected to it.
Automated scanners find newly opened ports within minutes.

⚠️ **Docker bypasses ufw.** `ufw` writes rules into `iptables`; Docker writes its
own rules and inserts them where they are evaluated first. A published port is
reachable regardless of firewall configuration, and `ufw status` will still
report it as blocked.

**Binding to `127.0.0.1` is not a firewall rule that could be bypassed — it is
the absence of a route.**

*`N8N_SECURE_COOKIE=false` is temporary, required only because the connection is
plain HTTP. To be removed in Phase 2 once TLS is in place.*

## 4. Reached it over an SSH tunnel

```bash
ssh -L 5678:localhost:5678 femi@SERVER_IP
# then http://localhost:5678 in the browser
```

**What this does.** SSH listens on port 5678 of the local machine and relays
anything arriving there, through the existing encrypted connection, to
`127.0.0.1:5678` as seen from the server — where Docker is listening.

**Only port 22 crosses the public internet.** The request travels inside the SSH
connection; port 5678 is never exposed at either end.

**The `localhost` in the middle argument is evaluated from the server's
perspective**, not the client's. Every machine has its own `127.0.0.1`, and
conflating the two is the most common source of confusion here.

This is the standard approach for reaching any service that should not be
public — databases, admin interfaces, internal dashboards.

## 5. Established where container data lives — by losing it

Created an account and a workflow in n8n, then:

```bash
docker stop n8n
docker start n8n     # data intact — stopping loses nothing
```

Then:

```bash
docker stop n8n
docker rm n8n
docker run ...       # identical image, command and name
```

**Result: setup screen. Account and workflow gone.**

**Cause.** A running container is two layers: the read-only image, and a writable
layer holding everything that changed since creation. All application state lived
in the writable layer. `docker rm` deletes it. A new container receives a fresh,
empty writable layer.

**This is the design, not a defect.** A container that accumulated state would be
as unique and irreproducible as a hand-configured server — which is the problem
containers exist to solve. Disposability is the property that makes them
interchangeable.

## 6. Added a volume, and proved persistence

```bash
docker volume create n8n_data

docker run -d \
  --name n8n \
  -p 127.0.0.1:5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e GENERIC_TIMEZONE="Africa/Lagos" \
  -e TZ="Africa/Lagos" \
  -e N8N_SECURE_COOKIE=false \
  n8nio/n8n
```

**A volume is a directory on the host, managed by Docker, mounted into the
container at a given path.** The container sees an ordinary folder; the data
physically resides on the host and outlives the container.

`/home/node/.n8n` is where n8n stores its database, credentials and workflows —
taken from the image's documentation. Identifying the persistent path is the
first thing to check when working with an unfamiliar image.

Confirmed the physical location:

```bash
docker volume inspect n8n_data
sudo ls -la /var/lib/docker/volumes/n8n_data/_data
# database.sqlite and config present
```

**Repeated the destruction test:**

```bash
docker stop n8n && docker rm n8n
docker volume ls                    # n8n_data still present
docker run -d ... -v n8n_data:...   # new container, different ID
```

**Account and workflow both present.** Verified the container ID differed from
the original, confirming it was genuinely a new container reading pre-existing
data.

⚠️ **The volume attaches to the container, not the image.** Every `docker run`
must specify `-v` again. Omitting it produces an empty application with the
volume sitting unused on disk.

## 7. Established what a volume is not

**A volume provides persistence. It does not provide protection.**

| | Protects against | Does not protect against |
|---|---|---|
| **Persistence** (volume) | Container deleted, crashed, upgraded | Host failure. Corruption. Accidental deletion. |
| **Redundancy** (live replica) | Disk or machine failure | Deletion — replicated within seconds |
| **Backup** (point-in-time copy, offsite) | All of the above | Nothing, if never tested by restoring |

**A volume holds current state, not history.** If data is corrupted or deleted,
the corruption persists — restarting the container achieves nothing, because the
container was never the problem.

**And the volume lives on the host's disk.** If the host is destroyed, the volume
goes with it. Genuine protection requires a copy held somewhere else entirely.

Backups and restore testing are Phase 3.

---

## Result

n8n running in a container, reachable only over an SSH tunnel, with application
state persisted in a Docker volume that survives container replacement.

## Next: the limitation this exposes

The working configuration is a seven-option command typed by hand. It is not
recorded in any file, cannot be placed under version control, and provides no
way to express startup ordering between multiple containers — which becomes
necessary as soon as a database is added.

Docker Compose addresses all three.

## Principles reinforced

**Bind to `127.0.0.1` unless a service genuinely needs public reach.** Do not
rely on a host firewall for container ports.

**Configuration belongs in Git. Data belongs in a volume.**

**Understand the security implication before accepting a convenience** — the
`docker` group being the example here.

**Prove behaviour rather than assuming it.** Destroying the container twice, with
and without a volume, established the boundary between what persists and what
does not far more reliably than reading about it.
