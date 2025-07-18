

# 1. Logical Volume Manager (LVM) Setup on AWS EC2

## Step 1: Create and Attach the EBS Volume

* Create a new **50 GB EBS volume** from the AWS Console.
* Attach it to your running EC2 instance (e.g., as `/dev/xvdh`).

## Step 2: Verify the Attached Volume

Run the following command to list all available block devices:

```bash
lsblk
```

Expected output:

```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0    8G  0 disk
└─xvda1 202:1    0    8G  0 part /
xvdh    202:80   0   50G  0 disk        <--- Unused and unmounted
```

---

## Step 3: Create a Partition with `fdisk`

Use `fdisk` to create a **single partition** on the new volume:

```bash
sudo fdisk /dev/xvdh
```

At the `Command (? for help):` prompt, follow this sequence:

1. `n` — Create new partition
2. Press `Enter` to accept default partition number
3. Press `Enter` to accept default first sector
4. Press `Enter` to use the entire available space
5. `t` — Change partition type
6. `8e` — Set type to "Linux LVM" (code 8e for MBR, **8e00 for GPT**)
7. `w` — Write changes

📌 **Note**: The default behavior **uses all available disk space** unless specified otherwise. That's why you press Enter for first and last sector prompts.

---

## Step 4: Verify the Partition Creation

```bash
lsblk
```

Expected output:

```
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda     202:0    0    8G  0 disk
└─xvda1  202:1    0    8G  0 part /
xvdh     202:80   0   50G  0 disk
└─xvdh1  202:81   0   50G  0 part        <--- New LVM partition
```

Look for a partition of type Linux LVM (code 8e or 8e00).

## Step 4: Create the Physical Volume (PV)
```
sudo pvcreate /dev/xvdh1
```

Output:
```
Physical volume "/dev/xvdh1" successfully created.
```

🔍 Verify PV Creation
```
sudo pvs
```

Expected:
```
PV         VG   Fmt  Attr PSize   PFree
/dev/xvdh1      lvm2 ---  50.00g  50.00g
```

Or:
```
sudo pvdisplay /dev/xvdh1
```

note: After you run pvcreate /dev/xvdh1, the lsblk output doesn't change much visually — the command lsblk shows the block devices, their partitions, and any existing filesystems, but it does not explicitly show that a partition is a physical volume (PV) for LVM.

## Step 5: Create the Volume Group (VG)
```
sudo vgcreate examplegroup1 /dev/xvdh1
```

Output:
```
Volume group "examplegroup1" successfully created
```

🔍 Verify VG Creation
```
sudo vgs
```

Expected:
```
VG             #PV #LV #SN Attr   VSize   VFree
examplegroup1   1   0   0  wz--n- 50.00g  50.00g
```

Or:
```
sudo vgdisplay examplegroup1
```

## Step 6: Create the Logical Volume (LV)
```
sudo lvcreate -n lvexample1 -L 9G examplegroup1
```

Output:
```
Logical volume "lvexample1" created
```

🔍 Verify LV Creation
```
sudo lvs
```

Expected:
```
LV          VG             Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
lvexample1  examplegroup1  -wi-a----- 9.00g
```

Or:
```
sudo lvdisplay /dev/examplegroup1/lvexample1
```

## Step 7: Create a Mount Point Directory
```
sudo mkdir /mnt1
```

🔍 Verify Directory Creation
```
ls -ld /mnt1
```

You should see:
```
drwxr-xr-x 2 root root 4096 ... /mnt1
```

## Step 8: Format the Logical Volume
```
sudo mkfs -t xfs /dev/examplegroup1/lvexample1
```

🔁 Use ext4, xfs, etc., depending on your preference.

🔍 Verify Filesystem Formatting
```
lsblk -f
```

Expected output includes:
```
/dev/examplegroup1/lvexample1 xfs UUID=...   <no mountpoint yet>
```

## Step 9: Mount the Filesystem
```
sudo mount /dev/examplegroup1/lvexample1 /mnt1
```

🔍 Verify Mount
```
df -h /mnt1
```

Expected:
```
Filesystem                        Size  Used Avail Use% Mounted on
/dev/mapper/examplegroup1-lvexample1   9G     ...   ...  /mnt1
```

## Step 10: Make the Mount Persistent
Edit /etc/fstab:
```
sudo nano /etc/fstab
```

Add this line:
```
/dev/examplegroup1/lvexample1 /mnt1   xfs     defaults,nofail   0   0
```

🔍 Verify Persistent Mount Works
Reboot your instance:
```
sudo reboot
```

After reboot:
```
df -h /mnt1
```
You should still see the volume mounted at /mnt1.

# -----------------------------------------------------------------------

# CGROUP

In Linux, a cgroup (short for control group) is a kernel feature that allows you to limit, prioritize, isolate, and monitor the usage of system resources — such as CPU, memory, I/O, and network — for a group of processes.

#### 🔍 In Simple Terms
Think of cgroups as a way to assign rules or quotas to a group of processes. For example:
* Limit a container or application to use only 2 GB of RAM.
* Prevent a background job from using more than 20% of CPU.
* Isolate disk I/O for different services.
* Monitor how much memory or CPU a process group is using.

#### 🧠 Why cgroups Matter
They're essential for:

| Use Case              | Example                                              |
| --------------------- | ---------------------------------------------------- |
| **Resource Limiting** | Limit an app to 1 CPU core and 512MB RAM             |
| **Isolation**         | Containers (e.g. Docker) use cgroups to isolate apps |
| **Monitoring**        | Track how much memory your services are using        |
| **Prioritization**    | Let a critical process get more CPU than others      |

#### 🛠️ Key Features of cgroups
- CPU: Limit or share CPU time between groups.
- Memory: Set hard or soft memory usage limits.
- BlkIO: Control disk I/O bandwidth.
- Network (via other tools): Indirectly manage network usage.
- PIDs: Limit the number of processes in a group.

#### 📁 Where to Find cgroup Info
On most systems:
```
cat /proc/cgroups
```

Mount points (depending on version):
- cgroups v1: /sys/fs/cgroup/
- cgroups v2: /sys/fs/cgroup/unified/ or just /sys/fs/cgroup/

##### Versions: cgroups v1 vs v2
| Feature               | cgroups v1              | cgroups v2             |
| --------------------- | ----------------------- | ---------------------- |
| Structure             | Multiple hierarchies    | Single unified tree    |
| Easier to manage?     | ❌ Complex               | ✅ Simplified           |
| Kernel support        | Older kernels (pre-4.x) | Kernel 4.x and above   |
| Used by default in... | CentOS 7, Ubuntu 16.04  | Ubuntu 20.04+, RHEL 8+ |

#####  Example: Limit Memory Using cgroups (v1)
```
mkdir /sys/fs/cgroup/memory/mygroup
echo 1000000000 > /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes
echo $$ > /sys/fs/cgroup/memory/mygroup/tasks
```
This limits the current shell to ~1 GB of RAM.

##### 🐳 cgroups and Containers
Tools like Docker, Kubernetes, systemd, and LXC all use cgroups under the hood to enforce resource limits and isolate workloads.





