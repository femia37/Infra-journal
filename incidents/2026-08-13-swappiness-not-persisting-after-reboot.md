# 2026-08-13 — swappiness reverted to default after reboot

## Symptom

Set `vm.swappiness` from the Ubuntu default of 60 down to 10. The change applied
successfully and was confirmed in the running system.

After a reboot, the value had reverted to 60.

```
$ cat /proc/sys/vm/swappiness
60
```

## Background

Swappiness controls how eagerly the kernel moves data out of RAM into swap
space. The default of 60 is tuned for desktop machines and is too eager for a
small server — it produces disk activity and slowdowns while RAM is still
available. A value of 10 means: prefer RAM, and reach for swap only when
genuinely short.

## What I checked

**1. Confirmed the current value:**
```bash
cat /proc/sys/vm/swappiness      # 60 — reverted
```

**2. Confirmed the setting had actually been written to a file:**
```bash
grep -n "swappiness" /etc/sysctl.conf
```
```
1:vm.swappiness=10
```

Present, at line 1, and not commented out. So the line existed and was
syntactically valid.

**3. Checked the surrounding file:**
```bash
tail -5 /etc/sysctl.conf
```

Returned only the single line. A stock `/etc/sysctl.conf` normally contains
extensive commented documentation, so the file had been overwritten rather than
appended to — a `tee` without `-a`. Harmless in this case, since everything lost
was comments, but worth noting as the append-versus-overwrite trap occurring for
real.

**4. Asked the system which files it actually reads:**
```bash
sudo sysctl --system
```

Output included:
```
* Applying /etc/sysctl.conf ...
...
vm.swappiness = 10
```

This ruled out the obvious explanations. The file *was* in the search path, the
syntax *was* valid, and no later file was overriding the value. Applied manually,
the setting worked correctly.

## Cause

The setting applied correctly when `sysctl --system` was run by hand, but was
not reliably picked up from `/etc/sysctl.conf` during boot.

`/etc/sysctl.conf` is the traditional location. Current Ubuntu reads settings
from files in `/etc/sysctl.d/`, with the old path continuing to work only
indirectly.

## Fix

Moved the setting to its own file in the directory the boot process reads
directly:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

No `-a` on `tee` here, deliberately — this creates a new file containing only
this setting, rather than appending to a shared file.

The `99-` prefix means it loads last, so it takes precedence over anything else.

**Rebooted a second time to verify:**
```bash
cat /proc/sys/vm/swappiness      # 10 — held
```

## Lesson

**Applying a setting and persisting a setting are two separate jobs.**

`sysctl` changes the value in the running system's memory. It never touches
disk. On restart, the system is rebuilt from files on disk — and if none of
those files mention the setting, it is gone.

The same pattern appears throughout Linux:

| Thing | Active now | Survives reboot |
|---|---|---|
| A service | `systemctl start X` | `systemctl enable X` |
| Swap | `swapon /swapfile` | a line in `/etc/fstab` |
| Kernel setting | `sysctl vm.swappiness=10` | a file in `/etc/sysctl.d/` |
| A container | `docker run` | `restart: unless-stopped` |

**The persistence step itself can silently fail.** Writing the setting to a file
is not the same as that file being read at the right moment. There was no error
message at any point — the only symptom was the value being wrong after a
restart.

**A persistence fix is not verified until you have rebooted and confirmed it.**
Re-running the command and seeing the right value proves nothing; it only
re-applies the temporary setting.

**When unsure, ask the machine rather than guessing.** `sysctl --system` prints
every file it reads as it reads them, which turned "I think this file is being
ignored" into a checkable fact.
