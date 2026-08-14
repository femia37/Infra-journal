# 2026-08-13 — "Text file busy" when creating a swap file

## Symptom

Attempting to create a 2 GB swap file on a freshly provisioned Ubuntu 26.04
server failed immediately:

```
$ sudo fallocate -l 2G /swapfile
fallocate: fallocate failed: Text file busy
```

## What I checked

```bash
swapon --show
```
```
NAME      TYPE SIZE USED PRIO
/swapfile file 2.3G 2.6M   -1
```

Swap was already active, backed by a file at exactly the path I was trying to
create.

```bash
ls -lh /swapfile
```
```
-rw------- 1 root root 2.4G Aug 13 11:08 /swapfile
```

The file existed, owned by root, with `600` permissions already correctly set.

```bash
cat /etc/fstab
```
```
/swapfile swap swap defaults 0 0
```

An entry was present in the filesystem table, meaning the swap was already
configured to activate automatically on every boot.

## Cause

The Vultr Ubuntu image ships with swap already created, activated, and made
persistent. Nothing needed to be built.

`fallocate` refused because the kernel was actively using that file as swap
space at the time. Reallocating a file while it is being written to as swap
would corrupt live memory data, so Linux blocks it. The error message states
this precisely: the file is busy.

## Fix

None required. The swap was already correct — right size, right permissions,
already persistent.

Only the swappiness value needed changing, which is a separate setting and is
covered in its own write-up.

## Lesson

**Check the current state before changing it.**

The information was already on screen. An earlier `free -h` had shown
`Swap: 2.3Gi` — it was read as a starting observation rather than as an answer
to a question that had not yet been asked.

The habit to build: before creating anything, verify whether it already exists.
`free -h` before adding swap. `systemctl status` before installing a service.
`ls -la` before creating a file.

**Read errors literally.** Linux error messages are terse but usually precise.
"Text file busy" means the file is busy. The instinct to build is to take the
message at face value and ask: *why would that be true?*

**Provider images differ.** Vultr configures swap by default; DigitalOcean does
not. Assuming either way is wrong — the correct move is to check.
