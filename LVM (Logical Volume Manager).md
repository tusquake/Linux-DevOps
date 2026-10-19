# LVM (Logical Volume Manager) in Linux

> **Who is this for?** A backend developer who has learned to attach a disk, format it, and mount it (Topic 10) and has now hit the limits of that approach: a filesystem is tied to **one fixed-size disk or partition**. LVM removes that limitation. It lets you **pool several disks together, carve out flexible volumes, and grow (or shrink) them whenever you like, often without downtime**. It is standard on enterprise Linux servers and a favorite interview topic.

---

## What is it and Why do we need it?

### What is LVM?

**LVM (Logical Volume Manager)** is a layer of software between your physical disks and your filesystems. Instead of formatting a disk directly, you give your disks to LVM, which combines them into one big pool of storage and lets you cut that pool into **logical volumes** of any size. Your filesystem lives on a logical volume rather than directly on a disk.

> **Analogy: Water Tanks, a Reservoir, and Taps**
> - Each physical disk is a **water tank**.
> - LVM connects all the tanks into **one big reservoir** (the pool).
> - From the reservoir, you run **separate pipes to different houses** (logical volumes), each getting as much water as you decide.
> - A house needs more water? **Open the valve wider** (extend the logical volume). The reservoir running low? **Connect another tank** (add a disk) and nobody has to turn off the taps.
>
> Without LVM, each house is hard-wired to exactly one tank. If the tank is too small, you must dig up the street.

### The problem LVM solves

With plain partitions (Topic 10), you run into these walls:

| Problem with plain partitions | How LVM handles it |
|-------------------------------|--------------------|
| A filesystem cannot be larger than its one disk or partition | A logical volume can **span multiple disks** |
| Growing a partition is awkward, and the free space must be right next to it | Add any new disk to the pool and extend the volume |
| Shrinking or reorganizing means copying data around by hand | Resize volumes, or **move data between disks live** (`pvmove`) |
| Backing up a busy filesystem consistently is hard | **Snapshots**: a frozen point-in-time view you can back up or roll back to |
| Replacing a failing disk means downtime | Migrate its data to another disk while the system runs |
| Space is wasted when each partition is sized by guesswork | Allocate what you need today, keep spare capacity in the pool, and assign it later |

### Why do we need it in DevOps?

- **Capacity changes constantly.** Databases and logs grow. With LVM, "we're running out of disk" becomes a two-minute online fix.
- **Cloud and bare metal.** In AWS, you can add another EBS volume to a pool and extend a mount point with no downtime. On physical servers, you can swap disks without rebuilding.
- **Standard on enterprise Linux.** The default installers on Red Hat, Rocky, and CentOS-family systems use LVM for the root filesystem, so you will meet it on real servers.
- **Safer changes.** Snapshots let you test risky upgrades with an instant undo button.

---

## Core Concepts

### 1. The Three Building Blocks: PV, VG, LV

```
     Physical world                                              What applications see
 +------------------+
 | Disk /dev/nvme1n1| --pvcreate--> [ PV ]---+
 +------------------+                        |
 +------------------+                        +--vgcreate--> [ VG: vg_data ] --lvcreate--> [ LV: lv_app ] --> /data
 | Disk /dev/nvme2n1| --pvcreate--> [ PV ]---+                    (pool)       \---------> [ LV: lv_logs ] --> /var/log/myapp
 +------------------+
```

| Block | Full name | What it is | Analogy |
|-------|-----------|-----------|---------|
| **PV** | **Physical Volume** | A disk or partition that has been **prepared for LVM** (it gets an LVM label and metadata) | An individual water tank |
| **VG** | **Volume Group** | A **pool** made from one or more PVs | The shared reservoir |
| **LV** | **Logical Volume** | A **slice of the pool** that behaves like a disk: you format it and mount it | A pipe feeding one house |

**The order is always: PV, then VG, then LV, then filesystem, then mount.**

---

### 2. Extents: The Unit of Allocation

LVM does not hand out space byte by byte. It divides the pool into equal-sized chunks called **extents**:

| Term | Meaning |
|------|---------|
| **PE (Physical Extent)** | A fixed-size chunk of a physical disk. The default size is **4 MiB** |
| **LE (Logical Extent)** | A chunk of a logical volume. Each LE maps to exactly one PE |

A logical volume is simply a **list of extents** that may sit on different disks. LV sizes are rounded to a multiple of the extent size.

> **Analogy:** Extents are **Lego bricks of equal size**. A logical volume is a structure built from bricks that may come from different boxes (disks). The structure looks seamless from the outside, but behind the scenes it is bricks from several places.

---

### 3. How LVM Appears in Linux

LVM uses the kernel's **device mapper** to present each logical volume as an ordinary block device. Every LV is reachable by two equivalent names:

| Path | Example |
|------|---------|
| `/dev/<vg_name>/<lv_name>` | `/dev/vg_data/lv_app` |
| `/dev/mapper/<vg_name>-<lv_name>` | `/dev/mapper/vg_data-lv_app` |

After creating an LV, you treat it exactly like a disk partition: `mkfs`, `mount`, add to `/etc/fstab`. Everything you learned in Topic 10 still applies; LVM just adds a flexible layer underneath.

---

### 4. LVM Features at a Glance

| Feature | Meaning | Typical use |
|---------|---------|-------------|
| **Resize (grow)** | Make an LV larger, usually online | Disk running out of space |
| **Resize (shrink)** | Make an LV smaller (needs care, and ext4 only) | Reclaim space for another volume |
| **Span disks** | One LV across multiple PVs | Exceed single-disk limits |
| **Snapshots** | Point-in-time, copy-on-write view of an LV | Consistent backups, safe upgrades with rollback |
| **pvmove** | Move extents from one PV to another, live | Replace or retire a disk without downtime |
| **Striping** | Spread data across disks for speed (like RAID 0) | Performance |
| **Mirroring / RAID** | Keep copies on several disks | Redundancy |
| **Thin provisioning** | Promise more space than physically exists, allocate on demand | Many volumes with uneven growth (needs monitoring) |

This guide focuses on the core features you will use every day: PV/VG/LV creation, extending, shrinking, snapshots, and `pvmove`.

---

### 5. Snapshots: How They Work

A **snapshot** is a frozen, point-in-time view of a logical volume. It does not copy all the data immediately. Instead it uses **copy-on-write (CoW)**:

1. The snapshot starts nearly empty. Reading the snapshot reads from the original volume.
2. When the **original** changes a block, LVM first **copies the old block** into the snapshot's reserved space, and then lets the change proceed.
3. So the snapshot always shows the data **as it was at the moment of creation**.

The snapshot needs a **reserved amount of space** for those saved old blocks. **If it fills up, the snapshot becomes invalid and is lost.**

> **Analogy:** Before the office repaints a wall, a photographer takes a photo. When a painter changes a section, they first photograph **just that section** as it used to look and put it in an album. To see the office as it was yesterday, you look at the original, except for the sections that are in the album. The album must be big enough for every section the painters change.

---

### 6. Important Cautions

| Caution | Explanation |
|---------|-------------|
| **LVM is not a backup** | Snapshots live in the same volume group on the same disks. If the disks fail, both are gone |
| **No redundancy by default** | A logical volume that spans two disks is **lost if either disk fails** (like RAID 0). Use mirroring, RAID, or rely on the cloud's durability and take regular backups (EBS snapshots, rsync to another host) |
| **Shrinking can destroy data** | Reducing an LV **below the size of the filesystem on it** cuts off data. Always shrink the **filesystem first** (or use `lvreduce -r`) |
| **XFS cannot shrink** | Only ext4 (and a few others) can be reduced |
| **Snapshots slow writes and can fill up** | Remove them when finished, and monitor their usage |
| **Complexity** | One more layer to understand during recovery |

---

### 7. LVM vs. Plain Partitions

| | Plain partition | LVM |
|---|-----------------|-----|
| Setup | Simpler | Slightly more steps |
| Grow later | Difficult; needs adjacent free space | Easy: add any disk to the pool |
| Span multiple disks | No | Yes |
| Shrink | Risky | Supported for ext4 (with care) |
| Snapshots | No | Yes |
| Move data live between disks | No | Yes (`pvmove`) |
| Best for | Fixed, simple layouts, the boot partition | Data volumes that grow or change |

---

## Practical Examples & Commands

> **Practice safely first.** Section 1 shows how to build a **lab with loop devices** (files that pretend to be disks), so you can run every LVM command without touching real disks or paying for cloud volumes. Sections 2 onward use realistic device names (`/dev/nvme1n1`, `/dev/nvme2n1`): **replace them with your own, confirmed with `lsblk`.**
>
> LVM commands that create or change storage need `sudo`. Install the tools if missing: `sudo apt install -y lvm2` (Debian/Ubuntu) or `sudo dnf install -y lvm2` (Red Hat family).

### 1. (Optional) Build a Safe Practice Lab with Loop Devices

```bash
sudo truncate -s 1G /tmp/disk1.img
sudo truncate -s 1G /tmp/disk2.img
# Create two sparse 1 GiB files that will pretend to be disks.

sudo losetup -f --show /tmp/disk1.img
sudo losetup -f --show /tmp/disk2.img
# Attach each file as a loop device. Each command PRINTS the device it used,
# for example /dev/loop10 and /dev/loop11. Use THOSE names wherever the examples say
# /dev/nvme1n1 and /dev/nvme2n1.

losetup -a
# List active loop devices.

lsblk
# They appear as disks of type "loop".
```

Clean-up for the lab (at the very end, after removing LVs, VGs, and PVs):

```bash
sudo losetup -d /dev/loop10 /dev/loop11     # Detach (use your device names)
sudo rm /tmp/disk1.img /tmp/disk2.img       # Delete the files
```

### 2. Inspect Before You Begin

```bash
lsblk -f
# Confirm the disks you plan to use are EMPTY: no FSTYPE, no MOUNTPOINT.
# pvcreate refuses disks that appear to contain data, but never rely on that: check yourself.

sudo pvs      # Short list of physical volumes (PVs)
sudo vgs      # Short list of volume groups (VGs)
sudo lvs      # Short list of logical volumes (LVs)
# On a fresh system with no LVM these print nothing.
# On a Rocky/RHEL server installed with defaults, you will see an existing "rl" or "centos" VG.

sudo lvmdiskscan
# Scan and list block devices that LVM can use.
```

### 3. Create Physical Volumes: `pvcreate`

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1
# Initialize two whole disks as PVs. (You can name several devices in one command.)
# Expected: "Physical volume "/dev/nvme1n1" successfully created."

sudo pvs
# Example output (sizes approximate):
#   PV             VG Fmt  Attr PSize   PFree
#   /dev/nvme1n1      lvm2 ---  <20.00g <20.00g
#   /dev/nvme2n1      lvm2 ---  <20.00g <20.00g
# The VG column is empty because the PVs do not belong to a pool yet.

sudo pvdisplay /dev/nvme1n1
# Detailed view: size, extent size, total and free extents, and the PV UUID.
```

> **Tip:** For EBS data volumes, using the **whole disk** as a PV is simple and makes later growth easy (see the `pvresize` example below). If you prefer partitions, create a partition first (Topic 10) and run `pvcreate` on the partition, for example `/dev/nvme1n1p1`.

### 4. Create a Volume Group: `vgcreate`

```bash
sudo vgcreate vg_data /dev/nvme1n1 /dev/nvme2n1
# Create a pool called "vg_data" from both PVs.
# Expected: "Volume group "vg_data" successfully created".

sudo vgcreate -s 8M vg_big /dev/nvme3n1
# -s sets a different physical extent size (default 4 MiB). Rarely needed.

sudo vgs
# Example output (sizes approximate):
#   VG      #PV #LV #SN Attr   VSize  VFree
#   vg_data   2   0   0 wz--n- 39.99g 39.99g
#   #PV = number of PVs, #LV = number of logical volumes, #SN = number of snapshots
#   VSize = total pool size, VFree = unallocated space

sudo vgdisplay vg_data
# Detailed pool information: extent size, total extents, allocated extents, free extents, UUID.
```

### 5. Create Logical Volumes: `lvcreate`

```bash
sudo lvcreate -L 25G -n lv_app vg_data
# Create a 25 GiB logical volume named lv_app inside the pool vg_data.
#   -L 25G   exact SIZE
#   -n       NAME

sudo lvcreate -L 5G -n lv_logs vg_data
# A second, separate 5 GiB volume from the same pool.

sudo lvcreate -l 100%FREE -n lv_rest vg_data
# -l (lowercase L) sizes by EXTENTS or percentages, instead of fixed units:
#   100%FREE  = all remaining free space in the pool
#   50%VG     = half of the whole pool
#   2560      = a number of extents (2560 x 4 MiB = 10 GiB)

sudo lvs
# Example output (extra columns trimmed):
#   LV      VG      Attr       LSize
#   lv_app  vg_data -wi-a----- 25.00g
#   lv_logs vg_data -wi-a----- 5.00g
# Attr "a" = active; later "o" (open) appears when mounted.

sudo lvdisplay /dev/vg_data/lv_app
# Detailed view, including the LV path, size, and current extents.

ls -l /dev/vg_data/ /dev/mapper/
# Both path styles exist.
```

### 6. Create a Filesystem, Mount It, and Persist It

```bash
sudo mkfs.ext4 /dev/vg_data/lv_app
# Format the LV just like any other disk. (Or: sudo mkfs.xfs /dev/vg_data/lv_app)

sudo mkdir -p /data
sudo mount /dev/vg_data/lv_app /data
df -hT /data
# Shows the filesystem at about 25G, with the device listed as /dev/mapper/vg_data-lv_app.

sudo chown -R appuser:appuser /data
# Give your application ownership (Topics 5 and 6).

# Persist across reboots (Topic 10): either the UUID or the stable LVM path works.
sudo blkid /dev/vg_data/lv_app
echo 'UUID=<paste-uuid-here>  /data  ext4  defaults,nofail  0  2' | sudo tee -a /etc/fstab
# Alternative line using the LVM path:
#   /dev/mapper/vg_data-lv_app  /data  ext4  defaults,nofail  0  2

sudo mount -a
# TEST fstab before rebooting. Fix any error immediately.
```

### 7. Extend a Logical Volume (The Reason LVM Exists)

**Case A: The pool still has free space**

```bash
sudo vgs
# Check the VFree column first.

sudo lvextend -L +10G -r /dev/vg_data/lv_app
# Grow the LV by 10 GiB AND resize the filesystem inside it in the same step.
#   -L +10G  ADD 10 GiB (the plus sign means "increase by"; without + it means "make it exactly this size")
#   -r       also resize the filesystem (works for ext4 and XFS, online)

sudo lvextend -l +100%FREE -r /dev/vg_data/lv_app
# Grow the LV by ALL remaining free space in the pool, and the filesystem with it.

df -h /data
# The mounted filesystem is now larger. No unmount, no downtime.
```

Without `-r` you must grow the filesystem as a separate step (this is the key thing people forget):

```bash
sudo lvextend -L +10G /dev/vg_data/lv_app     # Grow only the LV
sudo resize2fs /dev/vg_data/lv_app            # ext4: grow the filesystem to fill the LV
# or
sudo xfs_growfs /data                         # XFS: grow the filesystem (give the MOUNT POINT)
```

**Case B: The pool is full, so add a new disk to it**

This is the classic production scenario.

```bash
# 1. Attach a new disk (for example a new EBS volume, Topic 10). Suppose it appears as /dev/nvme3n1.
lsblk -f

# 2. Turn it into a PV
sudo pvcreate /dev/nvme3n1

# 3. Add the PV to the existing pool
sudo vgextend vg_data /dev/nvme3n1
sudo vgs
# VSize and VFree have grown by the size of the new disk.

# 4. Extend the logical volume and its filesystem
sudo lvextend -l +100%FREE -r /dev/vg_data/lv_app

# 5. Verify
df -h /data
```

**Case C: The underlying disk itself was enlarged (for example an EBS volume grown in AWS)**

```bash
# After increasing the EBS volume size in AWS (Topic 10):
lsblk                                   # The disk shows the new size
sudo pvresize /dev/nvme1n1              # Tell LVM that the PV got bigger
sudo vgs                                # VFree has increased
sudo lvextend -l +100%FREE -r /dev/vg_data/lv_app
# If the PV is a PARTITION, grow the partition first: sudo growpart /dev/nvme1n1 1
```

### 8. Shrink a Logical Volume (Use With Extreme Care)

> **Shrinking can destroy data.** Only ext4 can be shrunk. XFS cannot. **Take a backup first.** The data on the filesystem must already fit within the new size. The filesystem must be **shrunk before** the logical volume, never after.

Safe, automatic method (LVM handles the filesystem step for you):

```bash
sudo umount /data
# The filesystem must be unmounted to shrink ext4.

sudo e2fsck -f /dev/vg_data/lv_app
# Required consistency check before shrinking.

sudo lvreduce -r -L 15G /dev/vg_data/lv_app
# -r shrinks the filesystem FIRST, then the LV, to a final size of 15 GiB.
# (Using -L -10G would mean "reduce by 10 GiB".)

sudo mount /dev/vg_data/lv_app /data
df -h /data
sudo vgs
# VFree has increased by the space you released; assign it to another LV if you like.
```

Manual method (to understand what `-r` does):

```bash
sudo umount /data
sudo e2fsck -f /dev/vg_data/lv_app
sudo resize2fs /dev/vg_data/lv_app 15G       # 1) shrink the FILESYSTEM first
sudo lvreduce -L 15G /dev/vg_data/lv_app     # 2) THEN shrink the LV to the same size
sudo mount /dev/vg_data/lv_app /data
# Doing step 2 before step 1 cuts the end off the filesystem and corrupts data!
```

### 9. Snapshots: Safe Backups and Instant Rollback

```bash
sudo lvcreate -s -n lv_app_snap -L 5G /dev/vg_data/lv_app
# Create a snapshot of lv_app named lv_app_snap with 5 GiB reserved for changes.
#   -s   snapshot
#   -L   space reserved for copy-on-write data, NOT the size of the original
# The pool needs at least 5 GiB free.

sudo lvs
# Example output (extra columns trimmed). Watch the Data% column:
#   LV          VG      Attr       LSize  Origin  Data%
#   lv_app      vg_data owi-aos--- 25.00g
#   lv_app_snap vg_data swi-a-s--- 5.00g  lv_app  0.50
# Data% shows how much of the snapshot's reserved space is used up. NEAR 100% = DANGER.

sudo mkdir -p /mnt/snap
sudo mount -o ro /dev/vg_data/lv_app_snap /mnt/snap
# Mount the snapshot READ-ONLY. For XFS, also add nouuid:  -o ro,nouuid
# (An XFS snapshot has the same UUID as its original, and the kernel refuses duplicates.)

sudo tar -czf /backup/app-$(date +%F).tar.gz -C /mnt/snap .
# Back up a CONSISTENT, frozen view while the application keeps running.

sudo umount /mnt/snap
sudo lvremove /dev/vg_data/lv_app_snap
# Remove the snapshot as soon as you are done. It slows writes and can fill up.
```

**Roll back after a failed upgrade (merge the snapshot back):**

```bash
sudo systemctl stop myapp
sudo umount /data
# The original must not be in use for the merge to start immediately.

sudo lvconvert --merge /dev/vg_data/lv_app_snap
# Merges the snapshot back into the original, restoring the data to the moment of the snapshot.
# The snapshot volume is consumed (removed) once the merge completes.
# If the original was busy, the merge is deferred until the volume is next activated.

sudo mount /dev/vg_data/lv_app /data
sudo systemctl start myapp
```

### 10. Move Data Off a Disk, Live: `pvmove`

Scenario: `/dev/nvme1n1` is failing or must be retired, but its extents hold live data.

```bash
sudo pvs -o +pv_used
# See how much of each PV is in use.

sudo pvmove /dev/nvme1n1
# Move ALL allocated extents from nvme1n1 to the other PVs in the same volume group.
# This runs while the volumes stay mounted and in use. It needs enough free space on the other PVs.
# It can take a long time for large disks: run it in tmux or screen.

sudo pvmove /dev/nvme1n1 /dev/nvme2n1
# Or move specifically from one PV to another.

sudo vgreduce vg_data /dev/nvme1n1
# Remove the now-empty PV from the pool.

sudo pvremove /dev/nvme1n1
# Wipe the LVM label from the disk. It can now be detached or reused.
```

### 11. Rename, Remove, and Housekeeping

```bash
sudo lvrename vg_data lv_app lv_application      # Rename an LV (update fstab afterwards!)
sudo vgrename vg_data vg_prod                    # Rename a VG (update fstab and any mount configs)

sudo umount /data
sudo lvremove /dev/vg_data/lv_app                # DELETE a logical volume and its data (asks for confirmation)
sudo vgremove vg_data                            # Delete an empty volume group (remove its LVs first)
sudo pvremove /dev/nvme1n1 /dev/nvme2n1          # Remove LVM labels from disks

sudo vgchange -ay vg_data                        # Activate all LVs in a group (-an deactivates)
sudo lvchange -an /dev/vg_data/lv_app            # Deactivate one LV (it must be unmounted)

sudo vgscan && sudo lvscan                       # Re-scan for volume groups and logical volumes

sudo vgcfgbackup vg_data                         # Save the VG metadata to /etc/lvm/backup/
ls /etc/lvm/backup /etc/lvm/archive              # LVM automatically archives metadata before every change
# vgcfgrestore can restore it if the metadata is ever damaged. Keep these files safe.
```

### 12. Show Exactly What You Need: Reporting Options

```bash
sudo pvs -o pv_name,vg_name,pv_size,pv_free
sudo vgs -o vg_name,vg_size,vg_free,lv_count
sudo lvs -o lv_name,vg_name,lv_size,lv_path,devices
# "devices" shows which PVs actually hold each LV's data. Great for understanding the layout.

sudo lvs -a -o +devices
# Include hidden internal volumes (snapshots, metadata) and their devices.

lsblk
# Shows the LVM stack as a tree: disk, then lvm volume, then mount point.
```

### 13. Troubleshooting Cheat Table

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| `pvcreate`: "Device ... excluded by a filter" or "appears to contain a filesystem" | Disk already has data or a signature | Confirm it is the right and empty disk (`lsblk -f`), then `wipefs -a /dev/DEV` (DESTROYS data) and retry |
| `lvcreate`: "Volume group ... has insufficient free space" | The pool is smaller than requested | `vgs` to check `VFree`; add a disk with `vgextend`, or ask for less |
| `df -h` still shows the old size after `lvextend` | Filesystem not grown | `resize2fs` or `xfs_growfs`; use `lvextend -r` next time |
| `lvextend`: "Insufficient free space" | No free extents in the VG | `vgextend` with a new PV, or `pvresize` after enlarging a disk |
| `lvreduce` warns "WARNING: Reducing active logical volume ... may destroy your data" | You are shrinking the LV without shrinking the filesystem first | Cancel unless you have already shrunk the filesystem; use `lvreduce -r` |
| `resize2fs`: "Please run e2fsck -f first" | Filesystem was not freshly checked | Unmount, run `e2fsck -f`, retry |
| Snapshot shows `Data%` at 100% and becomes invalid | The reserved space was too small for the number of changes | Remove it; create new snapshots larger; keep them short-lived |
| LV missing after a reboot | VG or LV not activated, or the disk is missing | `sudo vgchange -ay`; check `lsblk` and `pvs` for missing PVs |
| Boot hangs after fstab change | Bad entry and no `nofail` | Use rescue mode or a rescue instance; fix `/etc/fstab` |
| `lvremove`: "Logical volume ... contains a filesystem in use" | Still mounted or held open | `umount`; check `lsof +f -- /data`, `fuser -vm /data` |

---

## Real-World Use Case

### Scenario: A growing database volume on AWS, built on LVM

A company runs a MySQL server on EC2. They expect fast growth and want to avoid late-night maintenance windows. They design the storage with LVM.

**Day 1: Build it**

```
   EC2 instance
   +----------------------------------------------------------+
   |  Root disk (8 GiB, ext4)                          /       |
   |                                                           |
   |  EBS vol A (100 GiB) ----+                                |
   |  EBS vol B (100 GiB) ----+--> VG: vg_db (200 GiB pool)    |
   |                                  |                        |
   |                                  +--> LV: lv_mysql (150 GiB)  --> /var/lib/mysql
   |                                  +--> LV: lv_backup (30 GiB)  --> /backup
   |                                  (20 GiB left unallocated for growth)
   +----------------------------------------------------------+
```

```bash
# Two EBS volumes were attached and appear as /dev/nvme1n1 and /dev/nvme2n1
lsblk -f

sudo pvcreate /dev/nvme1n1 /dev/nvme2n1
sudo vgcreate vg_db /dev/nvme1n1 /dev/nvme2n1

sudo lvcreate -L 150G -n lv_mysql  vg_db
sudo lvcreate -L 30G  -n lv_backup vg_db
# Notice we deliberately did NOT allocate the whole pool: spare capacity can be assigned later.

sudo mkfs.xfs -L mysql  /dev/vg_db/lv_mysql
sudo mkfs.xfs -L backup /dev/vg_db/lv_backup

sudo mkdir -p /var/lib/mysql /backup
cat <<'EOF' | sudo tee -a /etc/fstab
/dev/mapper/vg_db-lv_mysql   /var/lib/mysql  xfs  defaults,noatime,nofail  0  0
/dev/mapper/vg_db-lv_backup  /backup         xfs  defaults,nofail          0  0
EOF
sudo mount -a && df -hT /var/lib/mysql /backup
```

**Month 4: `/var/lib/mysql` reaches 85%: grow it online**

```bash
df -h /var/lib/mysql                 # 85% used
sudo vgs                             # 20 GiB still free in the pool

sudo lvextend -L +20G -r /dev/vg_db/lv_mysql
df -h /var/lib/mysql                 # Now 170 GiB, usage dropped, database never stopped
```

**Month 9: the pool itself is full: add a disk**

```bash
# Create and attach a new 200 GiB EBS volume (Topic 10); suppose it appears as /dev/nvme3n1
sudo pvcreate /dev/nvme3n1
sudo vgextend vg_db /dev/nvme3n1
sudo lvextend -l +80%FREE -r /dev/vg_db/lv_mysql      # Give most of the new space to the database
sudo vgs                                              # Some space remains for later needs
```

**Before a risky major version upgrade: snapshot for instant rollback**

```bash
sudo lvcreate -s -n lv_mysql_pre_upgrade -L 20G /dev/vg_db/lv_mysql
sudo lvs                                              # Watch Data% during the upgrade

# The upgrade goes wrong?
sudo systemctl stop mysql
sudo umount /var/lib/mysql
sudo lvconvert --merge /dev/vg_db/lv_mysql_pre_upgrade   # Roll back to the snapshot state
sudo mount /var/lib/mysql && sudo systemctl start mysql

# The upgrade succeeded? Discard the snapshot so it stops slowing writes:
sudo lvremove /dev/vg_db/lv_mysql_pre_upgrade
```

**Important safety net, not an afterthought**

LVM gives flexibility, **not protection**. The team also:

1. Takes regular **EBS snapshots** of each underlying volume and database-level backups copied off the host (`rsync` or object storage). **A volume group that spans two EBS volumes depends on both**; losing either loses the data in logical volumes that use it.
2. Alerts on `vgs` free space and on pool usage so they extend **before** reaching 90%.
3. Documents the exact extend procedure in a **runbook** and rehearses the restore path.

**Why this matters in production:**

- **No downtime for growth.** Extending storage becomes a routine, five-minute, online operation, not a scheduled outage.
- **Spare capacity in the pool** lets you react to surprises without ordering new hardware or rebuilding servers.
- **Snapshots** turn dangerous changes into reversible ones.
- **`pvmove`** lets you retire old or failing storage without ever stopping the service.

> **Analogy:** The reservoir approach means the town never has to dig up the street. When the database "house" needs more water, the operator opens a valve; when the reservoir runs low, they connect a new tank. Before major plumbing work, they photograph the pipes so they can restore the previous layout if something goes wrong.

---

## Interview Questions & Answers

### Q1. What is LVM, and what are its advantages over using standard partitions?

**Answer:**
LVM (Logical Volume Manager) adds an abstraction layer between physical disks and filesystems. Disks are combined into a pool (a volume group), from which flexible logical volumes are allocated. Advantages over plain partitions: logical volumes can be **resized online** (usually grown without unmounting), can **span multiple disks**, support **snapshots** for consistent backups and rollback, allow **live data migration** between disks (`pvmove`), and let you keep **spare capacity** in the pool to assign where needed. The trade-offs are extra complexity and the fact that LVM itself provides **no redundancy** unless combined with mirroring or RAID, so backups remain essential.

---

### Q2. Explain PV, VG, and LV, and the order of commands to set up LVM from scratch.

**Answer:**
A **Physical Volume (PV)** is a disk or partition initialized for LVM. A **Volume Group (VG)** is a pool made from one or more PVs. A **Logical Volume (LV)** is a slice of the pool that behaves like a disk. The setup order is:

1. `pvcreate /dev/nvme1n1 /dev/nvme2n1`: initialize the disks.
2. `vgcreate vg_data /dev/nvme1n1 /dev/nvme2n1`: create the pool.
3. `lvcreate -L 25G -n lv_app vg_data`: carve out a volume.
4. `mkfs.ext4 /dev/vg_data/lv_app`: create a filesystem.
5. `mkdir /data && mount /dev/vg_data/lv_app /data`: mount it, and add it to `/etc/fstab` with `nofail` (test with `mount -a`).

I verify at each step with `pvs`, `vgs`, `lvs`, and `lsblk`.

---

### Q3. The `/data` filesystem is full and the volume group has no free space. How do you add capacity without downtime?

**Answer:**
Attach a new disk (for example a new EBS volume in AWS), then:

1. `pvcreate /dev/nvme3n1` to prepare it as a PV.
2. `vgextend vg_data /dev/nvme3n1` to add it to the pool, then check with `vgs` that `VFree` increased.
3. `lvextend -l +100%FREE -r /dev/vg_data/lv_app` to grow the logical volume **and** the filesystem in one step. (`-r` calls the appropriate resize tool; otherwise run `resize2fs` for ext4 or `xfs_growfs <mountpoint>` for XFS separately.)
4. Verify with `df -h /data`.

All of this happens while the filesystem is mounted and in use. If the existing disk were simply enlarged instead (for example, an EBS resize), I would run `pvresize /dev/nvme1n1` (after `growpart` if it is partitioned) and then the same `lvextend -r`.

---

### Q4. Can you shrink a logical volume? What are the risks and the correct order of operations?

**Answer:**
Yes, but only for filesystems that support shrinking, such as **ext4**; **XFS cannot be shrunk**. The risk is **data loss**: if you reduce the logical volume before shrinking the filesystem inside it, the end of the filesystem is cut off and the data is corrupted. The correct order is: take a **backup**; **unmount**; run `e2fsck -f` on the volume; **shrink the filesystem first** (`resize2fs <lv> <newsize>`); **then reduce the LV** to the same size (`lvreduce -L <newsize> <lv>`); remount and verify. The safer shortcut is `lvreduce -r -L <newsize> <lv>`, which performs the filesystem shrink and the LV reduction in the correct order. The data must of course fit inside the new size.

---

### Q5. How do LVM snapshots work, and what should you watch out for?

**Answer:**
A snapshot is a point-in-time view of a logical volume, implemented with **copy-on-write**. Creating it is instant because nothing is copied up front; it just reserves space (`lvcreate -s -n snap -L 5G /dev/vg/lv`). When blocks on the original change, the **old blocks are copied into the snapshot's reserved space** first, so the snapshot always shows the data as of its creation. Uses: consistent backups of a live filesystem (mount the snapshot read-only and back it up), and quick rollback after a risky change (`lvconvert --merge`). Things to watch: if the reserved space **fills up (Data% reaches 100), the snapshot becomes invalid**; snapshots **slow down writes** on the origin, so they should be short-lived and removed afterwards (`lvremove`); XFS snapshots need `-o nouuid` when mounted alongside the original; and a snapshot is **not a real backup**, because it shares the same disks and volume group as the original.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| Install LVM tools | `sudo apt install -y lvm2` |
| Show PVs / VGs / LVs (short) | `sudo pvs`, `sudo vgs`, `sudo lvs` |
| Show details | `sudo pvdisplay`, `sudo vgdisplay`, `sudo lvdisplay` |
| Create a PV | `sudo pvcreate /dev/DEV` |
| Create a VG | `sudo vgcreate vg_name /dev/DEV1 /dev/DEV2` |
| Create an LV (fixed size) | `sudo lvcreate -L 25G -n lv_name vg_name` |
| Create an LV (all free space) | `sudo lvcreate -l 100%FREE -n lv_name vg_name` |
| Make a filesystem | `sudo mkfs.ext4 /dev/vg_name/lv_name` |
| Mount | `sudo mount /dev/vg_name/lv_name /mountpoint` |
| Add a disk to a VG | `sudo pvcreate /dev/DEV` then `sudo vgextend vg_name /dev/DEV` |
| Extend LV + filesystem | `sudo lvextend -L +10G -r /dev/vg_name/lv_name` |
| Use all free space | `sudo lvextend -l +100%FREE -r /dev/vg_name/lv_name` |
| Disk got bigger (cloud) | `sudo pvresize /dev/DEV`, then `lvextend -r` |
| Shrink LV (ext4 only) | `umount`, `e2fsck -f`, `lvreduce -r -L SIZE` |
| Create a snapshot | `sudo lvcreate -s -n snap -L 5G /dev/vg_name/lv_name` |
| Roll back to a snapshot | `umount`, `sudo lvconvert --merge /dev/vg_name/snap` |
| Remove a snapshot / LV | `sudo lvremove /dev/vg_name/NAME` |
| Move data off a disk | `sudo pvmove /dev/DEV`, then `vgreduce`, `pvremove` |
| Rename | `sudo lvrename vg old new`, `sudo vgrename old new` |
| Activate / deactivate | `sudo vgchange -ay vg_name`, `sudo lvchange -an LV` |
| Back up LVM metadata | `sudo vgcfgbackup` (files in `/etc/lvm/backup`) |

**Remember:**
- The chain is **PV, then VG, then LV, then filesystem, then mount**.
- Growing is **two layers**: the LV first, then the filesystem. `lvextend -r` does both.
- Shrinking is the reverse: **filesystem first, then LV** (or use `lvreduce -r`), ext4 only, and back up first.
- LVM provides **flexibility, not redundancy**. A snapshot is **not a backup**.
- `-L` sets **sizes** (`+10G` means "add 10G"), `-l` sets **extents or percentages** (`100%FREE`).
