# NTFS on Linux: Filesystems, Drivers, and fstab

---

## Filesystems: What They Actually Are

A filesystem is not the data on a disk — it is the system of rules that governs *how* data is stored, organized, and retrieved on a disk. Think of a raw disk as an enormous blank grid of storage blocks with no inherent structure. A filesystem imposes structure on that grid: it decides how to group blocks into files, how to name them, how to track which blocks belong to which file, how to handle permissions, timestamps, and so on.

Different operating systems have developed different filesystems, and they are not compatible with each other by default. Windows uses **NTFS** (New Technology File System), developed by Microsoft in the early 1990s as a replacement for the older FAT filesystem. Linux systems typically use **ext4**, **btrfs**, or **xfs**. macOS uses **APFS** or HFS+. Each of these speaks a completely different language at the disk level.

This is why plugging a Windows-formatted drive into a Linux machine is not automatic — Linux does not natively speak NTFS, at least not fully.

---

## The Linux Kernel and NTFS

The Linux kernel has had limited NTFS support for a long time — enough to read from NTFS volumes in basic cases, but writing was unreliable and unsupported for most of that history. This is because NTFS is a proprietary Microsoft format, and the kernel's built-in NTFS driver was written by reverse engineering the format rather than from any official specification.

In kernel version 5.15 (released 2021), a new in-kernel NTFS driver called `ntfs3` was merged, written by Paragon Software with Microsoft's cooperation. This driver provides proper read-write support directly in the kernel. However, it is relatively new and not always the default choice in practice.

The older and still widely used solution is `ntfs-3g` — a userspace driver that has been the standard approach for NTFS on Linux for well over a decade.

---

## What ntfs-3g Is

`ntfs-3g` stands for **NTFS Third Generation**. It is an open-source, full-featured NTFS driver that runs entirely in userspace rather than inside the kernel.

To understand what that means, it helps to understand the two layers of a running Linux system:

**Kernel space** is the privileged core of the OS — the part that directly controls hardware, manages memory, and handles low-level I/O. Drivers that run here have direct access to everything and operate at maximum speed.

**User space** is where all normal programs run — your terminal, browser, text editor, and so on. Programs here do not directly touch hardware; they ask the kernel to do it on their behalf.

Most filesystem drivers live in kernel space. `ntfs-3g` is different — it uses **FUSE** (Filesystem in Userspace), a Linux kernel interface that allows filesystem drivers to run as regular userspace programs. FUSE acts as a bridge: when Linux needs to read or write a file on an NTFS volume, the kernel passes that request through FUSE to the `ntfs-3g` process, which handles the NTFS logic and returns the result.

From the perspective of any program using the drive, this is completely invisible. Files appear, can be read and written, and behave normally. The FUSE layer handles all the routing transparently.

The tradeoff is a small performance overhead compared to in-kernel drivers, because every I/O operation crosses the kernel-to-userspace boundary twice. For an external USB drive, this overhead is negligible — the USB interface itself is far slower than the FUSE overhead.

### Installation

On Arch Linux:

```
sudo pacman -S ntfs-3g
```

On Debian/Ubuntu:

```
sudo apt install ntfs-3g
```

Installing the package makes the driver available to the system. It does not, by itself, mount anything — it simply gives Linux the capability to understand NTFS volumes when asked.

---

## Mounting: What It Means

In Linux, all storage — internal drives, external drives, USB sticks, network shares — is accessed through a single unified directory tree rooted at `/`. There are no drive letters like `C:\` or `D:\`. Instead, a drive's contents are made accessible by **mounting** it onto an existing directory. That directory is called the **mount point**.

When you mount a drive at `/mnt/external`, navigating to `/mnt/external` in a terminal or file manager shows the drive's contents. The directory itself is just a door — the drive is what's behind it.

Mounting can be done manually with the `mount` command:

```
sudo mount -t ntfs-3g /dev/sdb1 /mnt/external
```

This works, but requires you to know the device name (`/dev/sdb1`), specify the filesystem type, and run the command every time you want to use the drive. For drives you use regularly, this is where `/etc/fstab` comes in.

---

## /etc/fstab

`/etc/fstab` is a plain text configuration file — the **filesystem table** — that has been part of Unix systems since the 1970s. It tells Linux what to mount, where to mount it, what filesystem type it is, and what options to use. The kernel reads this file during boot and mounts everything listed in it (unless told otherwise).

Opening it reveals something like this:

```
# <device>              <mountpoint>         <type>     <options>              <dump> <pass>
UUID=abc123-...         /                    ext4       defaults               1      1
UUID=def456-...         /boot/efi            vfat       defaults               0      2
UUID=7CBE9AF7BE9AA962   /mnt/external        ntfs-3g    rw,uid=1000,nofail     0      0
```

Each non-comment line is one filesystem entry. There are six fields.

---

## The Six Fields of fstab

### Field 1: Device

Identifies which storage device or partition to mount. There are a few ways to specify this:

**By device name** — `/dev/sdb1`, `/dev/nvme0n1p3`, etc. Simple but fragile: device names are assigned dynamically at boot based on detection order. A drive that was `sdb1` today might be `sdc1` tomorrow if you have a different drive plugged in. Never use device names in fstab for external drives.

**By UUID** — `UUID=xxxxxxxx`. A UUID (Universally Unique Identifier) is a permanent identifier embedded in the partition itself when it was formatted. It never changes regardless of which port the drive is plugged into or what order devices are detected. This is the correct and standard approach.

```
lsblk -f          # shows filesystem type and UUID for all drives
blkid             # shows UUID, type, and label for all block devices (run as root)
```

Note that NTFS UUIDs look different from Linux filesystem UUIDs. Linux UUIDs follow the format `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` (lowercase, with hyphens). NTFS UUIDs are Windows-style: `7CBE9AF7BE9AA962` — uppercase hex, no hyphens. Both are valid in fstab.

**By label** — `LABEL=MyDrive`. Labels are human-readable names assigned to a partition. Convenient but not guaranteed to be unique — two drives could have the same label.

### Field 2: Mount Point

The directory in the filesystem tree where the drive's contents will appear. This directory must already exist before mounting. Common conventions:

- `/mnt/` — the traditional location for manually managed mounts. Persists across reboots.
- `/media/` — traditionally for removable media, managed automatically by desktop environments.
- `/run/media/<username>/` — the modern standard used by **udisks2**, the daemon that handles automatic USB mounting in desktop Linux environments. Note that `/run/` is a tmpfs (in-memory filesystem) that is wiped on every reboot, so mount point directories under it do not persist and may need to be recreated.

### Field 3: Filesystem Type

Tells the kernel what kind of filesystem is on the device, and therefore which driver to use. Common values:

| Value | Filesystem |
|-------|-----------|
| `ext4` | Standard Linux filesystem |
| `vfat` | FAT32, common for EFI and USB sticks |
| `ntfs-3g` | NTFS via the ntfs-3g userspace driver |
| `ntfs` | NTFS via the newer in-kernel ntfs3 driver |
| `btrfs` | Modern copy-on-write Linux filesystem |
| `auto` | Let the kernel detect the type automatically |

Writing `ntfs-3g` here is what connects the fstab entry to the driver you installed. If the package is not installed, mounting will fail with an error about an unknown filesystem type.

### Field 4: Options

A comma-separated list of mount options that control how the filesystem is mounted. This is the most detailed field and the one that requires the most understanding.

**General options:**

`defaults` — a shorthand that enables a standard set of options: `rw`, `suid`, `dev`, `exec`, `auto`, `nouser`, `async`. A reasonable baseline for most mounts.

`rw` — mount read-write. The drive can be read from and written to.

`ro` — mount read-only. No writes permitted. Useful for recovery or untrusted drives.

`auto` — mount this entry when `mount -a` is run (at boot or manually). This is the default behavior.

`noauto` — do not mount this entry automatically at boot or when `mount -a` is run. The entry exists in fstab (so its options are known), but mounting only happens when explicitly requested with `sudo mount /mountpoint`. This is appropriate for external drives that are not always connected.

`nofail` — if the device is not present, skip it silently and continue booting. Without this (or `noauto`), a missing drive causes the boot process to stall or drop to an emergency shell.

`noatime` — do not update the access timestamp on files when they are read. Normally, every file read triggers a write (to update the `atime` metadata field), which is wasteful on external drives. Disabling it improves performance with no practical downside for normal use.

`async` — filesystem I/O is done asynchronously. The kernel buffers writes and flushes them to disk in the background. Faster, but writes are not guaranteed to be on disk until explicitly flushed or the drive is unmounted. The default for most mounts.

`sync` — all writes go directly to disk before the write operation returns. Slower, but safer. Used for devices where data integrity is critical and sudden disconnection is a concern (some embedded systems, USB sticks).

**NTFS-specific options:**

`uid=1000` — assign ownership of all files on the volume to the user with UID 1000. NTFS has no concept of Linux user ownership, so this stamps every file with an owner at mount time. UID 1000 is almost always the first non-root user on a Linux system. Confirm your UID with `id -u`.

`gid=1000` — same as `uid` but for group ownership.

`umask=0022` — sets the default permission mask for **directories** on the volume. `0022` results in `755` permissions: the owner can read, write, and traverse directories; others can read and traverse but not write.

`fmask=0022` — sets the default permission mask specifically for **files**. `0022` results in `644`: owner can read and write, others can only read. `fmask` and `umask` are separate because executable bits on files need different treatment than on directories — keeping them separate avoids accidentally marking all files as executable.

`dmask` — an alternative way to set directory-specific permissions, equivalent to `umask` in ntfs-3g context.

### Field 5: Dump

A legacy field from the `dump` backup utility, which was commonly used in older Unix systems. Controls whether this filesystem should be included in dump-based backups. Almost always set to `0` (disabled) today — the `dump` utility is rarely used in modern Linux systems and has no effect if it is not installed.

### Field 6: Pass

Controls whether and when `fsck` (filesystem check) runs on this volume at boot.

- `0` — do not check. Use this for NTFS, FAT, and any non-Linux filesystem. `fsck` does not understand these formats and should never be run on them.
- `1` — check this filesystem first. Reserved for the root filesystem (`/`).
- `2` — check after the root filesystem. Used for other Linux partitions (`/home`, etc.).

For any NTFS entry, this must be `0`.

---

## What Happens at Boot

When Linux boots, one of the early steps is processing `/etc/fstab`. The system goes through each entry in order and, for entries without `noauto`, runs the equivalent of:

```
mount -t <type> -o <options> <device> <mountpoint>
```

For an ntfs-3g entry, this launches the `ntfs-3g` process, which connects to the FUSE kernel interface and begins serving filesystem requests for that mount point. The drive's contents become accessible at the specified path.

For entries with `noauto`, this step is skipped. The entry's existence in fstab still has value — it means when you later run `sudo mount /mnt/external`, the kernel already knows all the details (device, type, options) and does not need them specified on the command line.

---

## Manual Mounting with fstab in Place

Once an entry exists in fstab, mounting becomes much simpler. You can specify just the mount point and Linux looks up the rest:

```
sudo mount /mnt/external
```

This is equivalent to writing out the full command with all options — fstab is acting as a saved configuration.

After making any changes to fstab, test them immediately without rebooting:

```
sudo mount -a
```

This re-reads fstab and mounts everything that should be mounted. Any errors in your entries will surface here rather than during the next boot.

---

## Unmounting Safely

Before physically disconnecting an external drive, always unmount it:

```
sudo umount /mnt/external
```

Unmounting flushes any buffered writes to disk, releases the kernel's lock on the device, and terminates the `ntfs-3g` process associated with it. Pulling a drive without unmounting — especially if `async` is in effect — risks data corruption because buffered writes that have not yet reached the disk will be lost.

---

## Useful Reference Commands

```bash
lsblk -f                        # All drives, filesystems, UUIDs, mount points
blkid                           # UUIDs and types of all block devices (run as root)
cat /etc/fstab                  # View your current fstab
sudo mount -a                   # Mount all fstab entries (test after edits)
sudo mount /mnt/external        # Mount a specific entry using fstab config
sudo umount /mnt/external       # Unmount safely before disconnecting
dmesg | grep -i ntfs            # Kernel log messages related to NTFS
findmnt                         # Show all currently mounted filesystems in a tree
```

---

## A Complete Example Entry

```
UUID=7CBE9AF7BE9AA962   /mnt/external   ntfs-3g   rw,uid=1000,gid=1000,noatime,noauto,umask=0022,fmask=0022   0 0
```

Reading this left to right: mount the NTFS partition with this UUID, at `/mnt/external`, using the ntfs-3g driver, with read-write access, owned by UID/GID 1000, without updating access timestamps, not automatically at boot, with standard file and directory permission masks, no dump backup, no fsck check.

Every option is deliberate. Every field has a reason. This is the full picture of what a single line in `/etc/fstab` actually means.
