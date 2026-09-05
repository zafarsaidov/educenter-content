# Lesson 12: Disk Partitioning, File Systems & Disk Monitoring

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-11 (terminal through iptables/ufw/SSH — this closes out the Linux Basics module)

## Learning Objectives

By end of lesson student can:
- Explain block devices, and the difference between MBR and GPT partition tables
- Create and delete partitions with `cfdisk`, and describe what `parted` offers beyond it
- Format a partition with a filesystem (`mkfs.ext4`/`mkfs.xfs`) and mount/unmount it
- Write a correct `/etc/fstab` entry, including using a UUID instead of a device name
- Monitor disk usage and I/O with `df`, `du`, `lsblk`, `blkid`, `iostat`, `iotop`
- Create and enable swap space, and explain why swap matters

## Topics

- Disk concepts: block devices (`/dev/sda`, `/dev/vda`), MBR vs GPT partition tables
- `cfdisk`: create/delete partition, write; `parted` overview
- File systems: `mkfs.ext4`, `mkfs.xfs`; `mount`, `umount`; mount options
- `/etc/fstab`: fields (device, mountpoint, fstype, options, dump, pass), UUID
- Disk monitoring: `df -h`, `du -sh *`, `lsblk`, `blkid`; `iostat -x`, `iotop`
- Swap: `mkswap`, `swapon`/`swapoff`, `/etc/fstab` swap entry, `free -h`, `swapon --show`

## Concepts

### Block devices

A **block device** is how Linux represents a storage device the kernel can read/write in fixed-size chunks (blocks) — a physical disk, an SSD, or a virtual disk in a VM. Naming follows a convention: `/dev/sda` (first SCSI/SATA-style disk), `/dev/sdb` (second), etc.; `/dev/vda` on many virtualized/cloud instances (virtio disks). A number suffix identifies a partition on that disk: `/dev/sda1` is the first partition on `/dev/sda`.

### Partition tables: MBR vs GPT

A disk needs a **partition table** describing how it's divided into partitions before any filesystem can be created on it.

| | MBR (Master Boot Record) | GPT (GUID Partition Table) |
|---|---|---|
| Max partitions | 4 primary (or 3 primary + 1 extended holding logical partitions) | 128 by default |
| Max disk size | 2 TiB | Effectively unlimited (exabytes) |
| Redundancy | Single copy at start of disk | Primary + backup copy at end of disk |
| Modern default | Legacy, still supported | Standard for new systems |

GPT is the modern default for anything beyond a small legacy disk; MBR persists mostly for compatibility with older BIOS boot processes.

### Partitioning tools: `cfdisk` and `parted`

`cfdisk` is a simple, menu-driven, full-screen partitioning tool — good for straightforward create/delete/write workflows without memorizing command syntax. `parted` is a more powerful command-line tool that can also resize partitions, works non-interactively (scriptable), and supports both MBR and GPT directly. For a quick manual partition job, `cfdisk`'s interface is faster to work in; for scripted or advanced operations (resizing, precise sector control), `parted` is the better fit.

Critically: creating/writing a partition table is a **destructive** operation on whatever data already exists there. Both tools require an explicit "write" confirmation step precisely because this cannot be undone once written.

### Filesystems: ext4 and XFS

A partition is just raw block storage until formatted with a **filesystem** — the structure that organizes files, directories, and metadata (permissions, timestamps, the inode structures from Lesson 2) on top of the raw blocks.

| Filesystem | Notes |
|---|---|
| `ext4` | Default on most Ubuntu installs; mature, reliable, well-understood, good general-purpose choice |
| `xfs` | Excels at large files and high-throughput workloads (common in database/media storage use cases); harder to shrink than ext4 |

`mkfs.ext4 /dev/sdb1` and `mkfs.xfs /dev/sdb1` both format a partition with the respective filesystem — this is also destructive to anything already on that partition.

### Mounting

Formatting alone doesn't make a filesystem usable — it must be **mounted**: attached to a location in the existing directory tree (a **mount point**), after which files on that filesystem appear under that path. `mount /dev/sdb1 /mnt/data` makes `/dev/sdb1`'s contents accessible under `/mnt/data`; `umount /mnt/data` detaches it. A mount performed this way lasts only until reboot unless also recorded in `/etc/fstab`.

### `/etc/fstab`

`/etc/fstab` (filesystem table) tells the system which filesystems to mount automatically at boot. Each line has 6 whitespace-separated fields:

```
<device>        <mount point>   <fstype>   <options>       <dump>  <pass>
UUID=xxxx-xxxx  /mnt/data       ext4       defaults        0       2
```

| Field | Meaning |
|---|---|
| device | Which block device/partition — a UUID is strongly preferred over `/dev/sdb1`, since device letters can shift between boots (e.g. if a disk is added/removed) |
| mount point | Directory where it gets attached |
| fstype | `ext4`, `xfs`, etc. |
| options | Mount options, comma-separated (`defaults` covers the common sane defaults) |
| dump | Legacy backup-utility flag; `0` = ignore (nearly always 0 today) |
| pass | fsck check order at boot; `0` = never check, `1` = root filesystem, `2` = check after root |

`blkid` lists each device's UUID, which you copy into `/etc/fstab`. A typo or wrong UUID in `/etc/fstab` can, in the worst case, prevent a machine from booting cleanly — always test with `mount -a` (which re-reads `/etc/fstab` and mounts anything not already mounted) rather than rebooting blind.

### Disk monitoring tools

| Tool | Shows |
|---|---|
| `df -h` | Disk space used/free per mounted filesystem, human-readable |
| `du -sh *` | Disk space used per file/directory in the current location, summarized |
| `lsblk` | Tree view of block devices and their partitions/mount points |
| `blkid` | UUID and filesystem type per block device |
| `iostat -x` | Extended I/O statistics per device (throughput, wait time, utilization) |
| `iotop` | Live, per-process view of disk I/O (like `top`, but for disk activity) |

`df` answers "how full is this filesystem"; `du` answers "what inside this directory is taking up the space." They can disagree if a file is deleted while a process still has it open — `df` shows the space as still used (the blocks aren't freed until the last file handle closes) while `du` no longer sees the file at all, since it's unlinked from the directory tree.

### Swap

**Swap** is disk space the kernel uses as overflow when physical RAM is full — inactive memory pages get written out to swap, freeing RAM for active processes. It's much slower than RAM, so heavy swap usage is a performance red flag, but a modest amount of swap available prevents the kernel from being forced to kill processes outright (via the OOM killer) under memory pressure. `vm.swappiness` (Lesson 9) tunes how eagerly the kernel reaches for swap versus keeping things in RAM.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `lsblk` | Tree view of block devices | `lsblk` |
| `blkid` | Show UUID/fstype per device | `sudo blkid` |
| `cfdisk` | Interactive partition editor | `sudo cfdisk /dev/sdb` |
| `parted` | Scriptable partition editor | `sudo parted /dev/sdb print` |
| `mkfs.ext4` | Format a partition as ext4 | `sudo mkfs.ext4 /dev/sdb1` |
| `mkfs.xfs` | Format a partition as XFS | `sudo mkfs.xfs /dev/sdb1` |
| `mount` | Attach a filesystem | `sudo mount /dev/sdb1 /mnt/data` |
| `umount` | Detach a filesystem | `sudo umount /mnt/data` |
| `mount -a` | Mount everything in `/etc/fstab` | `sudo mount -a` |
| `df -h` | Disk space per filesystem | `df -h` |
| `du -sh` | Disk usage per file/dir | `du -sh /var/log/*` |
| `iostat -x` | Extended I/O stats | `iostat -x 2 5` |
| `iotop` | Live per-process disk I/O | `sudo iotop` |
| `mkswap` | Format a file/partition as swap | `sudo mkswap /swapfile` |
| `swapon`/`swapoff` | Enable/disable swap | `sudo swapon /swapfile` |
| `free -h` | Memory and swap summary | `free -h` |
| `swapon --show` | List active swap devices | `swapon --show` |

## Examples / Walkthrough

```bash
# --- inspecting existing disks ---
lsblk                                   # tree: disks, partitions, mount points, sizes
sudo blkid                              # UUID + filesystem type per device/partition

# --- partitioning a new disk (example: /dev/sdb, entirely unused) ---
sudo cfdisk /dev/sdb
# inside cfdisk: select free space -> [New] -> set size -> [Write] -> type "yes" to confirm -> [Quit]
# WARNING: writing a partition table destroys existing data on that disk

# parted equivalent, scriptable (example only — match to your actual disk/needs):
sudo parted /dev/sdb print               # show current partition table (or "unrecognised disk label")
sudo parted /dev/sdb mklabel gpt         # create a fresh GPT partition table (destructive)
sudo parted /dev/sdb mkpart primary ext4 0% 100%   # one partition using the whole disk

lsblk                                    # confirm the new partition (e.g. /dev/sdb1) now shows up

# --- formatting and mounting ---
sudo mkfs.ext4 /dev/sdb1                 # format with ext4
# or: sudo mkfs.xfs /dev/sdb1

sudo mkdir -p /mnt/data                  # create a mount point
sudo mount /dev/sdb1 /mnt/data           # mount it — lasts until reboot only
df -h /mnt/data                          # confirm it's mounted and see its size

sudo umount /mnt/data                    # detach it manually

# --- making the mount persistent via /etc/fstab ---
sudo blkid /dev/sdb1                     # copy the UUID shown here
sudo cat /etc/fstab                      # see existing entries first (root, boot, etc.)

# append a new line (using the UUID, not the device name, for stability across reboots):
echo 'UUID=<paste-uuid-here>  /mnt/data  ext4  defaults  0  2' | sudo tee -a /etc/fstab

sudo mount -a                            # re-read fstab and mount anything not already mounted
                                          # (safer test than rebooting — a typo here fails loudly, not at boot)
df -h /mnt/data                          # confirm

# --- disk usage monitoring ---
df -h                                    # overall disk space per mounted filesystem
du -sh /var/log/*                        # size of each item under /var/log, summarized
du -sh /home/* | sort -h                 # largest directories under /home (sort -h: human-readable sort)

iostat -x 2 5                            # extended I/O stats, every 2 seconds, 5 samples
sudo iotop                               # live view: which process is generating disk I/O right now

# --- swap ---
free -h                                  # current RAM and swap usage
swapon --show                            # currently active swap devices, if any

sudo fallocate -l 1G /swapfile           # allocate a 1GB file to use as swap
sudo chmod 600 /swapfile                 # restrict permissions — swap can contain sensitive memory contents
sudo mkswap /swapfile                    # format it as swap
sudo swapon /swapfile                    # activate it immediately

free -h                                  # confirm swap total increased
swapon --show                            # confirm /swapfile is listed

# persist across reboot
echo '/swapfile  none  swap  sw  0  0' | sudo tee -a /etc/fstab

sudo swapoff /swapfile                   # deactivate swap (e.g. before removing it)
```

## Common Pitfalls

- **Writing a partition table without triple-checking the target disk** — `cfdisk`/`parted` operate on whatever device path you give them; running against the wrong disk (e.g. `/dev/sda` instead of `/dev/sdb`) destroys the running system's own disk. Always confirm with `lsblk` first which device is actually the target.
- **Using `/dev/sdX` names in `/etc/fstab` instead of UUIDs** — device letters aren't guaranteed stable across reboots, especially if disks are added/removed or a VM's disk order changes. A UUID always identifies the same filesystem regardless of what name the kernel assigns it this boot.
- **A bad `/etc/fstab` entry breaking boot** — a wrong UUID, typo'd fstype, or unavailable device listed without a fallback option can hang or fail the boot process. Always test with `sudo mount -a` before rebooting, and know that most systems drop to an emergency shell/recovery prompt if `/etc/fstab` fails at boot, rather than looping forever.
- **Confusing `df` and `du` disagreement as a bug** — if `df` shows a filesystem nearly full but `du` can't find where the space went, it usually means a deleted file is still held open by a running process (common with log files that were deleted but not rotated properly). Restarting or signaling that process to reopen its log file frees the space.
- **Forgetting to `chmod 600` a swap file** — leaving default permissions on a swap file means other local users could potentially read it, and swap can contain sensitive data that was in RAM (env vars, credentials in memory).
- **Assuming XFS shrinks like ext4** — XFS filesystems can be grown but not shrunk. If there's a chance a partition will need to get smaller later, ext4 is the safer default choice.

## FAQ

**Q: When would I choose GPT over MBR today?**
A: Almost always GPT for any new disk — it supports disks larger than 2 TiB, more partitions, and has built-in redundancy. MBR is really only relevant for compatibility with very old BIOS-based boot processes.

**Q: What's the actual difference between `cfdisk` and `parted`?**
A: Same underlying job (managing partition tables), different interface. `cfdisk` is menu-driven and great for a one-off manual task. `parted` is command-driven, scriptable, and additionally supports resizing partitions — reach for it when automating or when you need finer control.

**Q: Why does `/etc/fstab` recommend UUIDs instead of `/dev/sdX`?**
A: Device names are assigned by detection order at boot, which isn't guaranteed stable — adding, removing, or reordering disks can shift which device gets which letter. A UUID is generated once when the filesystem is created and never changes, so it reliably points at the same filesystem regardless of device-naming order.

**Q: What happens if I mount a filesystem without adding it to `/etc/fstab`?**
A: It works immediately, but only until the next reboot or unmount — nothing persists the mount. Add an `/etc/fstab` entry (and test with `mount -a`) if the mount needs to survive a reboot.

**Q: Why does my system need swap if it has plenty of RAM?**
A: Even with ample RAM, some swap acts as a safety margin — it lets the kernel move rarely-used memory pages out of the way instead of being forced to kill a process outright when a temporary memory spike occurs. It's cheap insurance against an out-of-memory situation for a relatively small trade-off.

**Q: Is `iotop` different from `top`?**
A: Same live, refreshing interface style, but `iotop` shows disk I/O (read/write throughput) per process instead of CPU/memory — useful specifically when a machine feels slow due to disk activity rather than CPU load, which regular `top` won't clearly reveal.

## Practice / Exercise

**Core:**
1. On a lab VM with a spare unused disk (or an attached secondary virtual disk), run `lsblk` and identify the disk by its lack of partitions/mount points.
2. Use `cfdisk` to create a single partition spanning the whole disk, and confirm with `lsblk` afterward.
3. Format the new partition as `ext4`, create a mount point under `/mnt/`, and mount it manually.
4. Get the partition's UUID with `blkid`, write a correct `/etc/fstab` entry using it, and verify with `sudo mount -a` (after first `umount`-ing the manual mount).
5. Run `df -h` and `du -sh /var/log/*`; identify the largest item under `/var/log`.
6. Create a 512MB swap file, format and activate it, confirm with `free -h` and `swapon --show`, then deactivate and remove it.

**Stretch:**
1. Use `parted` non-interactively to print the partition table of a disk and explain each column of its output.
2. Deliberately write a wrong UUID into a *test* `/etc/fstab` entry (on a lab VM, never a production system) and observe/explain what `sudo mount -a` reports — don't reboot with the broken entry still in place; fix or remove it first.
3. Run `iostat -x 2 5` while copying a large file in another terminal, and identify which columns indicate the disk is under load.

## Further Reading

- `man fstab`, `man mount`, `man mkfs.ext4`, `man parted`, `man cfdisk`
- [Ubuntu Server Guide: Storage](https://ubuntu.com/server/docs/storage)
