# Linux Volume Management (AWS EBS Volumes, Mounting, Filesystem Creation)

> **Who is this for?** A backend developer whose application is starting to need **more disk space**, or whose database should not share a disk with the operating system. You have been using the single disk a server comes with. This guide teaches you how to **add a new disk, prepare it, attach it to the directory tree, make it survive reboots, and grow it later**, using AWS EBS as the real-world example.

---

## What is it and Why do we need it?

### What is volume management?

A **volume** is a chunk of storage that a server can use. In the cloud, you do not buy a physical hard drive. You **create a virtual disk with a few clicks**, **attach** it to your server, and then use Linux commands to make it usable. That whole process is volume management:

1. **Attach** a raw disk to the machine (cloud console or CLI).
2. **Detect** it in Linux (`lsblk`).
3. **Create a filesystem** on it, so Linux knows how to store files.
4. **Mount** it at a directory (such as `/data`), so you can read and write files there.
5. **Persist** the mount in `/etc/fstab`, so it comes back after a reboot.
6. **Grow** it later when you run low on space.

> **Analogy: Adding a New Warehouse to Your Business**
> - A **raw disk** is an **empty plot of land**. You can't store anything on bare ground.
> - A **filesystem** is **building the warehouse**: shelves, aisles, and a catalog so every box can be found.
> - A **mount point** is **the door that connects the new warehouse to your main building**. Walk through the `/data` door and you are inside the new warehouse.
> - **`/etc/fstab`** is the **standing instruction**: "Every morning when the company opens, re-connect that warehouse door."
> - **AWS EBS** is **renting the warehouse from a landlord** (AWS): you pick the size, pay for what you use, and the landlord can expand it later.

### Why do we need it?

| Need | How a separate volume helps |
|------|-----------------------------|
| **The disk is full** | Add or grow a volume without rebuilding the server |
| **Protect the OS** | If the app fills `/data`, the operating system on the root disk keeps running |
| **Data outlives the server** | Detach a data volume from a failed or replaced server and attach it to a new one |
| **Backups** | Snapshot one volume independently of the rest of the machine |
| **Performance** | Give a database a fast volume, and logs a cheap one |
| **Clean separation** | OS on one disk, application data on another: easier to reason about and secure |

---

## Core Concepts

### 1. The Storage Stack (From Hardware to Files)

```
   Application writes: /data/orders/2025.csv
                 |
   +-------------v--------------+
   |  Mount point:  /data       |   <- where it appears in the directory tree
   +-------------+--------------+
                 |
   +-------------v--------------+
   |  Filesystem: ext4 / xfs    |   <- how files are organized on the disk
   +-------------+--------------+
                 |
   +-------------v--------------+
   |  Partition (optional)      |   <- a slice of the disk
   |  /dev/nvme1n1p1            |
   +-------------+--------------+
                 |
   +-------------v--------------+
   |  Block device (the disk)   |   <- the raw volume
   |  /dev/nvme1n1              |
   +-------------+--------------+
                 |
   +-------------v--------------+
   |  AWS EBS volume (network   |   <- the actual storage in the data center
   |  attached storage)         |
   +----------------------------+
```

| Layer | What it is | Analogy |
|-------|-----------|---------|
| **Block device / disk** | A raw storage device with no structure yet | Bare land |
| **Partition** | A defined slice of a disk (optional) | A fenced plot on the land |
| **Filesystem** | The structure for storing and finding files | The warehouse's shelving and catalog |
| **Mount point** | The directory where a filesystem becomes accessible | The door into the warehouse |

A "block device" is a device that stores data in fixed-size blocks and allows reading or writing any block directly: hard drives, SSDs, and cloud volumes.

---

### 2. Device Names in Linux

All devices appear as files under `/dev`. The name depends on the type of disk:

| Name pattern | Meaning | Where you will see it |
|--------------|---------|-----------------------|
| `/dev/sda`, `/dev/sdb` | SCSI/SATA disks (first, second, ...) | Physical servers, VMs, some older cloud types |
| `/dev/xvda`, `/dev/xvdf` | Xen virtual disks | Older AWS instance types |
| `/dev/nvme0n1`, `/dev/nvme1n1` | NVMe disks (controller 0 namespace 1, controller 1 namespace 1, ...) | **Modern AWS instances (Nitro)**: EBS volumes appear as NVMe |
| `/dev/sda1`, `/dev/nvme0n1p1` | **Partition** 1 of a disk (`p` is inserted for NVMe) | Anything partitioned |

> **AWS gotcha:** When you attach a volume in the console you choose a device name like `/dev/sdf`. On modern (NVMe-based) instances, Linux **ignores that name** and shows the volume as `/dev/nvme1n1`, `/dev/nvme2n1`, and so on, in an order you should not rely on. **Always identify a disk by its size and `lsblk` output, and mount it by UUID (not by device name).**

---

### 3. Amazon EBS (Elastic Block Store) in Plain English

**EBS** provides **virtual hard drives for EC2 instances**. Key facts:

| Property | What it means |
|----------|---------------|
| **Network-attached** | The disk lives in AWS's storage system and is connected to your instance over the network, not physically inside it |
| **Persistent** | Data stays when the instance stops or restarts. (The *root* volume is deleted on termination by default; additional data volumes are kept unless configured otherwise) |
| **Tied to one Availability Zone (AZ)** | A volume in `us-east-1a` can only be attached to an instance **in `us-east-1a`** |
| **One instance at a time** (typical) | A standard volume attaches to one instance (a special Multi-Attach feature exists for certain types) |
| **Resizable (Elastic Volumes)** | Increase size, and change type, **while the volume is in use** |
| **Snapshots** | Point-in-time backups stored in S3-backed storage; **incremental** (only changed blocks after the first) |
| **Encryption** | Optional encryption at rest with KMS keys, transparent to Linux |

> **Analogy:** An EBS volume is an **external hard drive you plug into a laptop with a very long cable** that runs inside the data center. You can unplug it from one machine and plug it into another in the same building (AZ), but not across town.

**Common EBS volume types** (always check the AWS documentation for current specifications):

| Type | Kind | Typical use |
|------|------|-------------|
| **gp3** | General-purpose SSD (the modern default) | Boot volumes, databases, most workloads. Baseline of 3,000 IOPS and 125 MiB/s regardless of size, tunable upward |
| **gp2** | General-purpose SSD (older) | Legacy; performance tied to volume size |
| **io2 / io1** | Provisioned-IOPS SSD | Latency-sensitive, high-throughput databases |
| **st1** | Throughput-optimized HDD | Big sequential data: logs, data warehouses |
| **sc1** | Cold HDD | Rarely accessed, lowest cost |

**IOPS** = input/output operations per second: how many small reads/writes the disk can handle. **Throughput** = how many MB per second it can move.

**EBS vs. Instance Store:** Some instance types include **instance store** disks, which are physically attached, very fast, but **ephemeral**: their data is lost when the instance stops or fails. Use them only for caches and temporary data.

---

### 4. Partitions and Partition Tables

A **partition** is a defined slice of a disk. A **partition table** at the start of the disk records where each slice begins and ends.

| Partition table | Max disk size | Max partitions | Notes |
|-----------------|---------------|----------------|-------|
| **MBR** (older) | 2 TiB | 4 primary | Legacy |
| **GPT** (modern) | Practically unlimited | 128 by default | Recommended for new disks |

For **EBS data volumes**, partitions are **optional**. A very common and perfectly valid practice is to put the filesystem **directly on the whole disk** (`/dev/nvme1n1`), which makes growing it simpler. Partitioning is useful when you want multiple slices on one disk or the OS requires it (the root volume is always partitioned).

---

### 5. Filesystems

A **filesystem** defines how files, folders, names, permissions, and timestamps are stored on a disk.

| Filesystem | Characteristics | Typical use |
|------------|-----------------|-------------|
| **ext4** | Mature, reliable, the Ubuntu/Debian default. **Can grow and shrink** (shrinking requires unmounting) | General purpose |
| **XFS** | High performance with large files and parallel I/O; the default on RHEL, Rocky, Amazon Linux. **Can grow but cannot shrink** | Databases, big data, large volumes |
| **btrfs / zfs** | Advanced features (snapshots, checksums) | Specialist uses |

Related terms:

| Term | Meaning |
|------|---------|
| **Inode** | A record holding a file's metadata (owner, permissions, size, where its data lives). Each file uses one |
| **Journal** | A log of pending changes that lets the filesystem recover quickly after a crash |
| **UUID** | A unique ID string assigned to a filesystem when it is created. **It never changes**, unlike device names |
| **Label** | An optional human-friendly name for a filesystem, for example `data` |

> **Analogy:** ext4 and XFS are two **different shelving systems** for a warehouse. Both work well; XFS is built for huge, heavily used warehouses, ext4 is the dependable all-rounder. The **inodes** are the index cards in the catalog, and the **journal** is the logbook that lets staff pick up where they left off after a power cut.

---

### 6. Mounting, Mount Points, and `/etc/fstab`

**Mounting** attaches a filesystem to a directory. In Linux there is a **single directory tree**, and extra disks are grafted onto it at **mount points** (Topic 2). The directory becomes the entry point to that disk.

Two rules to remember:

1. The mount point **must already exist** as a directory (`mkdir /data`).
2. If the directory already has files, they are **hidden** (not deleted) while something is mounted on top of them, and reappear after unmounting. Always mount on an **empty** directory.

**A mount made by hand with `mount` disappears at reboot.** To make it permanent, add a line to `/etc/fstab` ("filesystem table"), which the system reads at boot.

An `/etc/fstab` line has six fields:

```text
UUID=3f6a9c2e-1b4d-4c7e-9a10-5d2e8b7f6a43  /data  ext4  defaults,nofail  0  2
        |                                    |      |         |          |  |
        |                                    |      |         |          |  +-- fsck order (0 = skip, 1 = root, 2 = others)
        |                                    |      |         |          +-- dump backup flag (almost always 0)
        |                                    |      |         +-- mount options
        |                                    |      +-- filesystem type
        |                                    +-- mount point
        +-- which filesystem (by UUID: stable and unambiguous)
```

Useful mount options:

| Option | Meaning |
|--------|---------|
| `defaults` | Standard set: read-write, executable, allows `mount -a`, and more |
| `nofail` | **Do not stop the boot if this disk is missing** (essential for cloud data volumes) |
| `noatime` | Do not update the "last accessed" time on every read (small performance gain) |
| `ro` | Mount read-only |
| `noexec` | Prevent running programs from this filesystem (a security hardening option) |
| `x-systemd.device-timeout=10` | Limit how long boot waits for the device |

> **Analogy:** `/etc/fstab` is a **standing-orders sheet for the building manager**: "Every morning, connect this warehouse (identified by its serial number) to the `/data` door, and **if that warehouse isn't there, still open for business** (`nofail`)."

> **Danger:** A mistake in `/etc/fstab` (wrong UUID, typo, missing `nofail`) can make the server **hang or drop into emergency mode at boot**, and on a cloud server you may lose SSH access. Always **test** with `sudo mount -a` before rebooting.

---

### 7. Growing Storage: Two Steps, Not One

Making a volume bigger always involves **two separate layers**:

| Step | Layer | Tool |
|------|-------|------|
| 1. Make the **disk** bigger | AWS EBS (console or CLI) | `aws ec2 modify-volume` |
| 2. Make the **partition** bigger (only if the disk is partitioned) | Linux | `growpart` |
| 3. Make the **filesystem** use the new space | Linux | `resize2fs` (ext4) or `xfs_growfs` (XFS) |

> **Analogy:** The landlord **expands the plot of land** (EBS). If you had fenced a plot inside it, you **move the fence** (partition). Finally you **extend the building** to fill the larger plot (filesystem). Until you extend the building, the extra land is unused.

Important limits: EBS volumes can only be **increased**, not decreased, and AWS limits how often one volume can be modified (roughly once every six hours). To go smaller, you create a new smaller volume and copy the data over.

---

## Practical Examples & Commands

> **Warning:** The commands that create filesystems (`mkfs`) and partitions **destroy existing data on the target device**. Triple-check the device name before pressing Enter. Practice on a new, empty, disposable volume.

### 1. Inspect Disks and Mounts

```bash
lsblk
# List block devices as a tree. The most useful storage command.
# Example output (yours will differ):
#   NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
#   nvme0n1      259:0    0    8G  0 disk
#   +-nvme0n1p1  259:1    0    7G  0 part /
#   +-nvme0n1p15 259:2    0  106M  0 part /boot/efi
#   nvme1n1      259:3    0   20G  0 disk             <-- the NEW, unused 20 GB volume
# A disk with no MOUNTPOINT and no partitions is not in use yet.

lsblk -f
# Adds FILESYSTEM information: FSTYPE (ext4, xfs...), LABEL, UUID, and mount point.
# An EMPTY FSTYPE for a disk means it has no filesystem yet: safe to format.

lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINT
# Choose exactly which columns to show.

sudo blkid
# Print the UUID, filesystem type, and label of every block device.

sudo file -s /dev/nvme1n1
# Look at the first bytes of a device.
#   "data"                       -> empty, no filesystem (safe to format)
#   "SGI XFS filesystem data..." or "Linux ... ext4 filesystem data..." -> ALREADY has a filesystem: DO NOT format!

sudo fdisk -l
# List all disks and their partition tables in detail.

df -h
# Show MOUNTED filesystems: size, used, available, and where each is mounted.

df -hT
# Same, plus the filesystem TYPE.

findmnt
# Show all mounts as a tree. Use "findmnt /data" to examine one mount point.

sudo nvme list
# On NVMe-based AWS instances: map /dev/nvmeXn1 to the EBS volume ID (needs the nvme-cli package).

ls -l /dev/disk/by-uuid/
# Stable names for each filesystem (symlinks pointing to the real devices).
```

### 2. Create and Attach an EBS Volume (AWS)

**Using the AWS Console:**

1. Open **EC2**, then **Elastic Block Store**, then **Volumes**, then **Create volume**.
2. **Volume type:** `gp3`. **Size:** for example `20` GiB.
3. **Availability Zone:** **must be the same AZ as your instance** (visible in the instance's details).
4. Optionally enable **Encryption** and add tags. Click **Create volume**.
5. Select the new volume, then **Actions**, then **Attach volume**. Choose your instance and a device name such as `/dev/sdf`. Click **Attach**.

**Using the AWS CLI** (needs credentials with permission to manage EC2 volumes):

```bash
# 1. Create a 20 GiB gp3 volume in the same AZ as the instance
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 20 \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=app-data}]'
# The output includes the new VolumeId, for example vol-0abc123def4567890

# 2. Wait until it is available
aws ec2 wait volume-available --volume-ids vol-0abc123def4567890

# 3. Attach it to the instance
aws ec2 attach-volume \
  --volume-id vol-0abc123def4567890 \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf

# 4. Check its state ("in-use" means attached)
aws ec2 describe-volumes --volume-ids vol-0abc123def4567890 \
  --query 'Volumes[0].[VolumeId,State,Size,VolumeType,Attachments[0].InstanceId]'
```

### 3. Prepare the New Volume Inside Linux (Whole-Disk Method)

This is the standard, simplest approach for EBS data volumes. It assumes the new disk appeared as `/dev/nvme1n1` (confirm with `lsblk`).

```bash
# STEP 1: Identify the new disk. It should be the unmounted disk with the expected size.
lsblk -f

# STEP 2: Verify it is EMPTY before you format it
sudo file -s /dev/nvme1n1
# Expect:  /dev/nvme1n1: data

# STEP 3: Create a filesystem
sudo mkfs.ext4 /dev/nvme1n1
# Format the whole disk as ext4. (Equivalent: sudo mkfs -t ext4 /dev/nvme1n1)
# Or, for XFS:
# sudo mkfs.xfs /dev/nvme1n1
#
# Add a label if you like:  sudo mkfs.ext4 -L appdata /dev/nvme1n1

# STEP 4: Create the mount point (an empty directory)
sudo mkdir -p /data

# STEP 5: Mount it
sudo mount /dev/nvme1n1 /data

# STEP 6: Verify
df -hT /data                      # Shows the new filesystem, its size, and type
lsblk -f                          # The disk now shows FSTYPE and MOUNTPOINT /data

# STEP 7: Give your application user ownership (permissions: Topics 5 and 6)
sudo chown -R appuser:appuser /data
sudo chmod 750 /data

# STEP 8: Smoke test
echo "hello volume" | sudo tee /data/test.txt
cat /data/test.txt
```

### 4. Make the Mount Permanent with `/etc/fstab`

```bash
# STEP 1: Get the filesystem UUID
sudo blkid /dev/nvme1n1
# Example:  /dev/nvme1n1: UUID="3f6a9c2e-1b4d-4c7e-9a10-5d2e8b7f6a43" TYPE="ext4"

# STEP 2: Always back up fstab before editing it
sudo cp /etc/fstab /etc/fstab.bak

# STEP 3: Add the entry (replace the UUID and filesystem type with YOUR values)
echo 'UUID=3f6a9c2e-1b4d-4c7e-9a10-5d2e8b7f6a43  /data  ext4  defaults,nofail  0  2' | sudo tee -a /etc/fstab
# For XFS use "xfs" as the type and 0 as the last field:
#   UUID=...  /data  xfs  defaults,nofail  0  0

# STEP 4: TEST the file WITHOUT rebooting
sudo umount /data                 # Unmount it by hand
sudo mount -a                     # Mount everything listed in fstab
# If "mount -a" prints an error, FIX IT NOW. Do not reboot with a broken fstab.
df -h /data                       # Confirm it came back

findmnt --verify
# Check /etc/fstab for syntax problems (available on modern systems).

# STEP 5: (Optional) The ultimate test: reboot and confirm the mount survives
sudo reboot
# After reconnecting:  df -h /data
```

### 5. Use a Partition Instead (When You Need One)

```bash
sudo parted -s /dev/nvme1n1 mklabel gpt mkpart primary ext4 0% 100%
# Non-interactive: create a GPT partition table, then ONE partition using 0% to 100% of the disk.

lsblk
# Shows the new partition: nvme1n1p1

sudo mkfs.ext4 /dev/nvme1n1p1
# Create the filesystem on the PARTITION (note the p1), not on the whole disk.

sudo mkdir -p /data && sudo mount /dev/nvme1n1p1 /data
```

The interactive alternative with `fdisk`:

```bash
sudo fdisk /dev/nvme1n1
# Inside fdisk, type:
#   g       create a new GPT partition table
#   n       new partition (press Enter to accept the defaults for number, start, and end)
#   p       print the table to review it
#   w       write the changes and exit (nothing is saved until you press w)
```

### 6. Mount Options, Labels, and Unmounting

```bash
sudo mount -o ro /dev/nvme1n1 /mnt
# Mount read-only (for inspection or recovery).

sudo mount -o noatime,nodev,nosuid /dev/nvme1n1 /data
# Performance and hardening options on the command line.

mount | grep /data
# Show how it is mounted, including the options in effect.

sudo e2label /dev/nvme1n1 appdata
# Set or change the label on an ext4 filesystem. (XFS: sudo xfs_admin -L appdata /dev/nvme1n1, while unmounted.)

sudo umount /data
# Unmount a filesystem by its mount point (note: umount, not "unmount").

sudo umount /dev/nvme1n1
# ...or by device.
```

If `umount` says **"target is busy"**, something is using it:

```bash
sudo lsof +f -- /data
# List processes with open files on that filesystem.

sudo fuser -vm /data
# Show the processes using the mount.

cd ~
# A very common cause: your own shell is currently INSIDE /data. Move out and try again.

sudo systemctl stop myapp
# Stop the service that uses the data, then unmount.

sudo umount -l /data
# "Lazy" unmount: detaches now and cleans up once it is no longer busy (a last resort).
```

### 7. Check and Repair Filesystems

```bash
sudo umount /data
sudo fsck -n /dev/nvme1n1
# Check an UNMOUNTED ext4 filesystem for errors. -n = report only, change nothing.

sudo fsck -f /dev/nvme1n1
# Force a full check and repair when you are sure the filesystem is unmounted.

sudo xfs_repair -n /dev/nvme1n1
# The XFS equivalent (-n = no modify, dry run). XFS uses xfs_repair, not fsck.
```

> **Never run `fsck` on a mounted filesystem.** It can corrupt data. Unmount first.

### 8. Grow a Volume Online (No Downtime)

Scenario: `/data` (20 GiB) is 90% full. Grow it to 40 GiB while the application keeps running.

```bash
# STEP 1 (AWS): Enlarge the EBS volume
aws ec2 modify-volume --volume-id vol-0abc123def4567890 --size 40
# (Or in the Console: Volumes, select it, Actions, Modify volume, new size.)

aws ec2 describe-volumes-modifications --volume-ids vol-0abc123def4567890 \
  --query 'VolumesModifications[0].[ModificationState,Progress]'
# Wait until the state is "optimizing" or "completed". You can continue at "optimizing".

# STEP 2 (Linux): Confirm the OS sees the larger DISK
lsblk
# nvme1n1 now shows 40G, but the filesystem (df -h) still shows about 20G.

# STEP 3 (Linux): Grow the FILESYSTEM
# Case A: ext4 directly on the whole disk
sudo resize2fs /dev/nvme1n1

# Case B: XFS directly on the whole disk (give the MOUNT POINT, not the device)
sudo xfs_growfs -d /data

# STEP 4: Verify
df -h /data
# Now about 40G. No unmount, no reboot, no downtime.
```

If the volume is **partitioned** (this includes the **root volume**), grow the partition first:

```bash
sudo apt install -y cloud-guest-utils
# Provides "growpart". (On Amazon Linux/RHEL: sudo dnf install -y cloud-utils-growpart)

sudo growpart /dev/nvme0n1 1
# Grow partition number 1 of /dev/nvme0n1 to fill the disk.
# NOTE THE SPACE between the disk name and the partition number.

lsblk
# The partition now fills the disk.

sudo resize2fs /dev/nvme0n1p1          # ext4 root (typical on Ubuntu)
# or
sudo xfs_growfs -d /                   # XFS root (typical on Amazon Linux / RHEL)

df -h /
```

### 9. Snapshots: Back Up and Clone a Volume

```bash
aws ec2 create-snapshot \
  --volume-id vol-0abc123def4567890 \
  --description "app-data before upgrade" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=app-data-pre-upgrade}]'
# Take a point-in-time backup. The first snapshot copies everything; later ones store only changes.
# For consistency on a busy database, pause writes (or use the database's own backup tools) first.

aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[*].[SnapshotId,VolumeId,State,StartTime]' --output table

# Restore: create a NEW volume from the snapshot (in any AZ), then attach and mount it
aws ec2 create-volume --snapshot-id snap-0123456789abcdef0 \
  --availability-zone us-east-1b --volume-type gp3
```

> A volume restored from a snapshot **already contains the filesystem**. **Do not run `mkfs` on it** (that would erase the restored data). Just attach it and `mount` it.

### 10. Move a Volume to Another Instance (Recovery and Migration)

```bash
# On the OLD instance
sudo systemctl stop myapp
sudo umount /data

aws ec2 detach-volume --volume-id vol-0abc123def4567890
aws ec2 wait volume-available --volume-ids vol-0abc123def4567890

# Attach to the NEW instance (it must be in the SAME Availability Zone)
aws ec2 attach-volume --volume-id vol-0abc123def4567890 \
  --instance-id i-0fedcba9876543210 --device /dev/sdf

# On the NEW instance: DO NOT run mkfs. The data is already there.
lsblk -f                              # Find it and note the UUID
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data         # Mount it
ls /data                              # Your files are present
# Then add the fstab line (see example 4).
```

The same technique rescues a server that won't boot: attach its **root volume** to a healthy "rescue" instance as a secondary disk, mount it, and fix the configuration files (for example a broken `/etc/fstab`).

### 11. Copy Data to a New Volume (Adding a Volume to Existing Data)

```bash
# You want to move /var/lib/postgresql onto the new volume mounted at /data
sudo systemctl stop postgresql
sudo rsync -aHAX /var/lib/postgresql/ /data/postgresql/
# -a archive, -H preserve hard links, -A preserve ACLs, -X preserve extended attributes (Topic 7)

# Mount the new volume at the ORIGINAL path, instead of a new path, by editing fstab:
#   UUID=...  /var/lib/postgresql  ext4  defaults,nofail  0  2
# Keep the old directory contents until you have verified the new setup.
```

### 12. Clean Up (Avoid Surprise Bills)

```bash
sudo umount /data                       # 1. Unmount
sudo nano /etc/fstab                    # 2. Remove or comment the /data line
aws ec2 detach-volume --volume-id vol-0abc123def4567890              # 3. Detach
aws ec2 wait volume-available --volume-ids vol-0abc123def4567890
aws ec2 delete-volume --volume-id vol-0abc123def4567890              # 4. Delete (permanent!)
# Unattached volumes and old snapshots continue to cost money until deleted.
```

### 13. Troubleshooting Cheat Table

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| New disk not visible in `lsblk` | Volume not attached, or in a different AZ | Check the volume state and AZ in AWS; run `lsblk` again |
| `mount: wrong fs type, bad option, bad superblock` | No filesystem on the disk yet, or wrong type specified | `lsblk -f`; run `mkfs` if it is truly empty |
| `mount: ... can't find in /etc/fstab` | No mount point or entry given | Specify both device and directory |
| `mount point does not exist` | Directory missing | `sudo mkdir -p /data` |
| `target is busy` on `umount` | Open files or your shell is inside the mount | `lsof +f -- /data`, `fuser -vm /data`, `cd ~` |
| Server hangs at boot after editing fstab | Bad entry and no `nofail` | Detach the volume or use a rescue instance, fix `/etc/fstab` |
| `df -h` shows the old size after growing | Filesystem not resized yet | `resize2fs` or `xfs_growfs` |
| `growpart: no space left on device` (or `NOCHANGE`) | Partition already fills the disk, or the OS has not noticed the new size | Check `lsblk`; confirm the EBS modification finished |
| `Permission denied` writing to `/data` | New filesystems are owned by root | `chown` and `chmod` (Topics 5 and 6) |
| Disk is "full" but `du` shows less used | Out of inodes, or a deleted file still held open | `df -i`; `sudo lsof \| grep deleted` (Topic 4) |
| Files vanished after mounting | Mounted over a non-empty directory | `umount`; the originals reappear |

---

## Real-World Use Case

### Scenario: A PostgreSQL server that runs out of disk space, fixed without downtime

A startup's production database runs on one EC2 instance. Originally, everything lived on the single 8 GiB root disk. Over time, the team made two improvements, and then dealt with a real incident.

**Improvement 1: Separate the database data from the operating system**

```
   EC2 instance (us-east-1a)
   +------------------------------------------------+
   |  Root volume   8 GiB gp3    mounted at  /      |   OS, packages, logs
   |  Data volume  50 GiB gp3    mounted at  /data  |   PostgreSQL data directory
   +------------------------------------------------+
            |                          |
       snapshot daily            snapshot hourly
```

```bash
# 1. Create and attach a 50 GiB gp3 volume (same AZ): see example 2
# 2. Format it with XFS (a good fit for databases) and mount it
sudo mkfs.xfs -L pgdata /dev/nvme1n1
sudo mkdir -p /data
UUID=$(sudo blkid -s UUID -o value /dev/nvme1n1)
echo "UUID=$UUID  /data  xfs  defaults,noatime,nofail  0  0" | sudo tee -a /etc/fstab
sudo mount -a && df -hT /data

# 3. Move the database onto it (during a maintenance window)
sudo systemctl stop postgresql
sudo rsync -aHAX /var/lib/postgresql/ /data/postgresql/
# Point PostgreSQL's data_directory at /data/postgresql in its configuration, then:
sudo systemctl start postgresql
```

**Why this design helps:** if a runaway log or query fills `/data`, the **operating system on `/` keeps working**, so you can still log in and fix things. The data volume can be **snapshotted on its own schedule**, resized independently, and **moved to a replacement server** if the instance fails.

**The incident: 6 months later, an alert fires: `/data` is 88% full**

```bash
df -hT /data
# /dev/nvme1n1  xfs  50G  44G  6.0G  88%  /data

du -sh /data/postgresql/* | sort -h | tail -n 5       # What is using the space? (Topics 3, 4)

# The data is legitimate growth. Take a safety snapshot first, then grow online.
aws ec2 create-snapshot --volume-id vol-0abc123def4567890 --description "pre-resize"
aws ec2 modify-volume --volume-id vol-0abc123def4567890 --size 100

# Wait for "optimizing", then grow the filesystem (XFS takes the MOUNT POINT):
lsblk                                  # Disk now 100G
sudo xfs_growfs -d /data
df -hT /data                           # 100G total, now about 44% used

# The database never stopped. Applications saw no downtime.
```

**Follow-up actions the team takes:**

1. Add a **monitoring alert** at 80% disk usage so they have time to act (rather than at 98%).
2. Adopt a **runbook**: snapshot, modify volume, grow filesystem, verify.
3. Record that EBS volumes **cannot shrink** and are limited in how often they can be modified, so they size generously with headroom, and use gp3's ability to tune IOPS and throughput separately from size.
4. Test the **restore path** by creating a volume from a snapshot, attaching it to a scratch instance, and verifying the database starts.

**Why this matters in production:**

- Disk-full incidents are among the **most common causes of outages**. Knowing the safe online-resize procedure turns a crisis into a five-minute routine.
- Separating data from the OS is a standard **resilience pattern**: it contains the blast radius, simplifies backup, and speeds recovery.
- `nofail` and UUID-based fstab entries prevent the "server won't boot after a reboot" surprise that locks teams out of cloud instances.

> **Analogy:** The company's main building (root volume) stays clean and operational. The **inventory warehouse** (data volume) was built separately. When the warehouse is 88% full, the landlord **expands the plot** and the staff **extend the building** while the shop stays open. Nobody has to close for the day, and every night a photographer takes a snapshot of the warehouse in case of fire.

---

## Interview Questions & Answers

### Q1. What is the difference between a disk, a partition, a filesystem, and a mount point?

**Answer:**
A **disk** (block device, such as `/dev/nvme1n1`) is the raw storage. A **partition** is an optional slice of a disk with its own device name (`/dev/nvme1n1p1`), defined in the partition table (MBR or GPT). A **filesystem** (ext4, XFS) is the on-disk structure that organizes files, directories, permissions, and metadata; without one you can't store files. A **mount point** is the directory in the Linux tree (such as `/data`) where a filesystem is attached so that applications can access it. The order is: disk, then optionally partition, then filesystem (`mkfs`), then mount (`mount`).

---

### Q2. Walk me through adding a new EBS volume to a running Linux instance so that it survives reboots.

**Answer:**
1. **Create** the EBS volume in the **same Availability Zone** as the instance, choosing size and type (for example gp3), and **attach** it.
2. In Linux, find it with `lsblk` (modern instances show it as `/dev/nvme1n1`, whatever device name was requested).
3. Verify it is empty with `sudo file -s /dev/nvme1n1` (output "data") or `lsblk -f`.
4. **Create a filesystem:** `sudo mkfs.ext4 /dev/nvme1n1` (or `mkfs.xfs`).
5. **Make a mount point and mount:** `sudo mkdir /data && sudo mount /dev/nvme1n1 /data`.
6. **Persist:** get the UUID with `sudo blkid`, add `UUID=<uuid> /data ext4 defaults,nofail 0 2` to `/etc/fstab`, then test with `sudo umount /data && sudo mount -a`.
7. Set **ownership and permissions** for the application user.

---

### Q3. Why should `/etc/fstab` use the UUID instead of a device name like `/dev/sdb`, and what is `nofail` for?

**Answer:**
Device names are assigned by the kernel at boot in the order devices are detected, so they **can change** between reboots (especially when volumes are added or removed, or on NVMe where AWS-requested names are ignored). A **UUID is generated when the filesystem is created and stays the same**, so it always identifies the right disk. `nofail` tells the system **not to treat a missing or failing disk as a boot failure**. Without it, a missing volume can drop the server into emergency mode, which on a cloud instance may mean you cannot SSH in. I always test fstab changes with `mount -a` before rebooting.

---

### Q4. A volume mounted at `/data` is almost full. How do you increase its size without downtime? Why can't you simply "shrink" it later?

**Answer:**
First increase the EBS volume with `aws ec2 modify-volume --size <new>` (or in the console), then wait for the modification to reach the "optimizing" state. In Linux, confirm the new size with `lsblk`. If the disk is **partitioned**, grow the partition with `growpart /dev/nvmeXn1 <partition-number>`. Then grow the **filesystem**: `resize2fs <device>` for ext4, or `xfs_growfs <mount point>` for XFS. This all works while the filesystem is mounted and in use. EBS volumes can only be **increased**, never decreased, and **XFS cannot be shrunk** at all. To go smaller you must create a new smaller volume, copy the data, and switch over. Because of that, and because there is a limit on how often a volume can be modified, I plan capacity with headroom and monitor usage with alerts.

---

### Q5. What are the key characteristics of EBS, and how does it differ from instance store?

**Answer:**
EBS is **network-attached, persistent block storage**: data survives instance stops and restarts, volumes can be **detached and re-attached** to another instance **in the same Availability Zone**, resized online, **snapshotted** (incrementally, for backup and cloning), and encrypted with KMS. There are several types (gp3 for general use, io2 for high-performance databases, st1/sc1 for throughput or cold data). **Instance store** is storage physically attached to the host: very fast, but **ephemeral**: data is lost when the instance stops, terminates, or the hardware fails, so it is suitable only for caches, buffers, and scratch data. Also note that the root EBS volume is deleted on termination by default, while additional data volumes persist unless configured otherwise.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| List disks and partitions | `lsblk`, `lsblk -f` |
| Filesystem UUIDs | `sudo blkid` |
| Is a device empty? | `sudo file -s /dev/nvme1n1` (output "data" means empty) |
| Map NVMe to EBS volume ID | `sudo nvme list` |
| Disk usage of mounted filesystems | `df -hT` |
| Create ext4 / XFS filesystem | `sudo mkfs.ext4 /dev/DEV`, `sudo mkfs.xfs /dev/DEV` |
| Create a GPT partition | `sudo parted -s /dev/DEV mklabel gpt mkpart primary ext4 0% 100%` |
| Create mount point and mount | `sudo mkdir /data`, `sudo mount /dev/DEV /data` |
| Unmount | `sudo umount /data` |
| Who is blocking the unmount? | `sudo lsof +f -- /data`, `sudo fuser -vm /data` |
| Persist across reboots | Add `UUID=... /data ext4 defaults,nofail 0 2` to `/etc/fstab` |
| Test fstab safely | `sudo mount -a`, `findmnt --verify` |
| Check ext4 / XFS (unmounted) | `sudo fsck -n /dev/DEV`, `sudo xfs_repair -n /dev/DEV` |
| Grow EBS volume (AWS) | `aws ec2 modify-volume --volume-id ID --size N` |
| Grow partition | `sudo growpart /dev/DEV 1` |
| Grow ext4 / XFS filesystem | `sudo resize2fs /dev/DEV`, `sudo xfs_growfs -d /mountpoint` |
| Snapshot a volume | `aws ec2 create-snapshot --volume-id ID` |
| Attach / detach volume | `aws ec2 attach-volume ...`, `aws ec2 detach-volume ...` |

**Remember:**
- **Same AZ** for the volume and the instance.
- **`mkfs` erases data.** Never run it on a volume that already holds data (including volumes restored from snapshots).
- **UUID plus `nofail`** in fstab, and **test with `mount -a`** before rebooting.
- Growing is **two layers**: the EBS volume first, then the filesystem (and the partition if there is one).
- EBS volumes **grow but never shrink**.
