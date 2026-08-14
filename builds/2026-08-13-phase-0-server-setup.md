# Phase 0 — Initial server setup and hardening

**Date:** 2026-08-13
**Machine:** Vultr Cloud Compute · Shared CPU · 1 GB RAM · Johannesburg
**OS:** Ubuntu 26.04 LTS

---

## Goal

Take a freshly provisioned cloud server from its default state to one that is
safe to expose to the public internet and ready to host services that run
continuously.

Every cloud provider hands over a machine with root login enabled and password
authentication switched on, because they need some way to give you initial
access. That is a starting state, not a recommendation. The work below closes it.

---

## 1. Provisioned the instance

Chose deliberately:

| Choice | Reason |
|---|---|
| Johannesburg region | Closest datacentre to Lagos — lower latency for daily SSH work |
| Ubuntu LTS | Long Term Support: five years of security patches, and the widest pool of documentation when troubleshooting |
| $5/mo tier, not $2.50 | The cheaper plan is IPv6-only, which would make the server unreachable from most consumer connections |
| Skipped the provider's SSH key option | Installing the key manually is how the permission model is actually learned |
| Skipped Marketplace pre-built images | Installing software by hand is the point |
| Auto-backups off | The server will be deliberately destroyed and rebuilt as an exercise |

Located the provider's browser-based console **before** starting any changes.
That console connects directly to the machine and bypasses SSH entirely, so it
remains available if SSH access is broken. Knowing where it is in advance turns
a lockout from a crisis into an inconvenience.

## 2. Updated the system

```bash
apt update && apt upgrade -y
```

A server image is built at some point in the past, so parts of it are already
out of date on first boot. `apt update` refreshes the catalogue of available
software; `apt upgrade` installs the newer versions.

## 3. Created a non-root user

```bash
adduser femi
usermod -aG sudo femi
```

The machine arrived with only the `root` account, which can do anything
including deleting the operating system while it is running. The risk is not
deliberate destruction — it is that root receives no error for a mistyped
command. It simply executes it.

Standard practice is to work as a normal user and borrow root's powers one
command at a time via `sudo`.

The `-a` in `-aG` means append. Without it the command would *replace* every
group the user belongs to, stripping permissions they need.

**Verified in a second terminal window before proceeding**, keeping the original
root session open as a fallback:

```bash
ssh femi@SERVER_IP
sudo whoami        # returned: root
```

## 4. Installed SSH key authentication

Copied the existing public key from the laptop into the server:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Passwords are short pieces of text and can be found by a machine making enough
attempts. A private key cannot — the number of possibilities is beyond what any
computer could work through.

The permissions are not optional. SSH silently refuses to use a key file that
other users on the machine could read or modify, because they could otherwise
add their own key to it and log in as you.

Tested from a fresh terminal tab before changing anything else.

## 5. Hardened SSH

**Checked the provider override directory first:**

```bash
ls -la /etc/ssh/sshd_config.d/
```

Found `50-cloud-init.conf` containing `PasswordAuthentication yes`, left by the
provider's first-boot configuration tool. The main config file begins with
`Include /etc/ssh/sshd_config.d/*.conf`, and in SSH the *first* setting read
wins — so this file overrides anything written in the main config.

Missing this is a common cause of "I disabled password login but it still works."

**Settings applied:**

| Setting | Value | Effect |
|---|---|---|
| `PermitRootLogin` | `no` | Root cannot log in over SSH at all |
| `PasswordAuthentication` | `no` | Passwords are not accepted from anyone |
| `PubkeyAuthentication` | `yes` | Key authentication explicitly enabled |

Blocking root matters more than it first appears: every Linux machine has an
account called `root`, so an attacker already knows a username that definitely
exists. Blocking it means they must now guess both a username and a credential.

**Found and resolved a duplicate.** `PermitRootLogin` appeared twice — `no` at
line 54 and `yes` further down, appended by the provider. The `no` was already
winning under SSH's first-wins rule, but contradictory security settings are how
mistakes happen later, so the second was commented out with a note explaining why.

**Tested the configuration before applying it:**

```bash
sudo sshd -t                  # silence = valid syntax
sudo systemctl restart ssh    # existing sessions survive a restart
```

**Verified from a new connection while the old ones stayed open:**

```bash
ssh femi@SERVER_IP     # succeeded, no password prompt
ssh root@SERVER_IP     # Permission denied (publickey)
```

That second message is the success condition — the server listing `publickey`
as the only method it will consider, with password absent from the list entirely.

## 6. Configured the firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw show added        # confirmed OpenSSH rule present
sudo ufw enable
```

**Deny-by-default rather than blocking known-bad traffic**, because you can only
block what you already know about. If software later starts listening on an
unexpected port, a deny-by-default policy has already stopped it.

**Rules before enabling, deliberately.** Enabling first would apply the deny-all
policy to the existing SSH session immediately, ending it mid-command. Recovery
would require the provider's console. This is the most common way people lock
themselves out of a server.

Ports 80 and 443 opened ahead of need, for HTTP and HTTPS in a later phase. An
open port with nothing listening behind it simply refuses connections — the port
is not the risk, what listens behind it is.

## 7. Verified swap and tuned swappiness

**Checked existing state first** rather than assuming:

```bash
free -h
```

Found 2.3 GB of swap already configured by the provider's image. No swap file
needed to be created. (Providers differ on this — DigitalOcean images ship with
none.)

Reduced swappiness from the default 60 to 10:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

Swap is disk space used as overflow when RAM fills, preventing the kernel's
OOM killer from terminating running processes without warning. Because it is
disk rather than memory, it is much slower — a safety net, not extra capacity.

Swappiness controls how eagerly the kernel moves data out to that space. The
default of 60 is tuned for desktop machines and is too eager for a small server,
producing slowdowns while RAM is still available. 10 means: prefer RAM, and use
swap only when genuinely short.

*(This step produced two incidents — see `incidents/`.)*

## 8. Enabled automatic security updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Vulnerabilities are published publicly when discovered, and automated scanners
begin probing for them within hours. A server patched a week late was exposed
for a week.

Security updates only, not feature updates — so nothing restarts unexpectedly,
but security holes close quickly.

## 9. Rebooted and verified

```bash
sudo reboot
```

The reboot is the actual test. Applying a setting and persisting a setting are
two separate jobs, and plenty of configurations work perfectly until the first
restart.

Verified after restart:

- [x] Key login works, no password prompt
- [x] `ssh root@IP` → `Permission denied (publickey)`
- [x] `ufw status` → active, three rules
- [x] Swap present
- [x] Swappiness = 10
- [x] `unattended-upgrades` active
- [x] Kernel update applied, restart-required notice cleared

## 10. Secured the control plane

Enabled two-factor authentication on the provider account.

Everything above protects the server from attack over the network. The provider
account sits above all of it — from that panel, whoever holds the account can
reinstall the OS, snapshot the disk, open a root console bypassing SSH, or
destroy the machine. No amount of server hardening addresses that.

---

## Result

A publicly-reachable server where the only route in is a private key file held
on one laptop, with no password authentication offered, no reachable root
account, all incoming connections dropped except three deliberately opened
ports, and security patches applying automatically.

## Principles applied

**Never cut the branch you are sitting on.** When changing anything that
controls your own access: know the escape route first, keep a working session
open, test the configuration before applying it, and verify from a fresh
connection before closing the old ones.

**Check state before changing state.** Confirm a thing does not already exist
before building it.

**A valid config is not a correct config.** Syntax checks verify grammar, not
intent — every setting could be commented out and still pass. Confirm settings
actually took effect afterwards.

**Applying and persisting are separate jobs.** Ask of every change: will this
still be true after a reboot? Then restart and confirm.
