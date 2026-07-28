# How to Create a RAID 5 Array with mdadm
This guide walks through creating a Linux software RAID 5 array using mdadm. It assumes you're comfortable with the terminal and have at least three empty disks available.

Example disks used throughout: /dev/sdb, /dev/sdc, and /dev/sdd

Replace these with your actual device names.

# Before You Begin
RAID 5 warning
RAID 5 provides redundancy, not a backup.

# Keep these points in mind:
Creating a RAID array erases all data on the selected disks.
RAID 5 can survive one disk failure.
If a second disk fails before the first is rebuilt, the array is lost.
Large modern drives can take many hours (or even days) to rebuild, increasing the risk of another failure during that time.
Always keep important data backed up somewhere else.

# Step 1 — Install mdadm
- Debian/Ubuntu:
sudo apt update
sudo apt install mdadm

- RHEL, Rocky, AlmaLinux, Fedora:
sudo dnf install mdadm

This installs the Linux software RAID management tools.

# Step 2 — Identify Your Drives
- List all storage devices:
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT

or

sudo fdisk -l

- Make absolutely sure you've identified the correct disks.

- Example:
/dev/sdb
/dev/sdc
/dev/sdd

# Step 3 — Verify the Drives Are Not Mounted
- Check:
lsblk

- If any partitions are mounted:
sudo umount /dev/sdb1
sudo umount /dev/sdc1
sudo umount /dev/sdd1

- Replace the partition names with your own if necessary.

# Step 4 — (Optional but Recommended) Wipe Existing RAID Metadata
- If the disks have been used before:
sudo mdadm --zero-superblock /dev/sdb /dev/sdc /dev/sdd

- If the command reports there was no RAID metadata, that's fine.

# Step 5 — Create the RAID 5 Array
sudo mdadm \
  --create /dev/md0 \
  --level=5 \
  --raid-devices=3 \
  /dev/sdb /dev/sdc /dev/sdd

- Explanation:
/dev/md0 is the new RAID device.
--level=5 creates RAID 5.
--raid-devices=3 tells mdadm how many disks are in the array.

- You'll be asked for confirmation.
Type:

yes

# Step 6 — Watch the Array Build
- Immediately check its status:
cat /proc/mdstat

- Example:
md0 : active raid5 sdb[0] sdc[1] sdd[2]

- The array will begin syncing in the background.

- You can continue with formatting while it syncs, though performance may be reduced until initialization completes. Monitor progress with /proc/mdstat until it reports the array is fully synchronized. 


# Step 7 — Verify Array Details
sudo mdadm --detail /dev/md0

- This displays:
RAID level
member disks
UUID
rebuild status
health

# Step 8 — Create a Filesystem
- Format the new array with ext4:
sudo mkfs.ext4 /dev/md0

- This prepares the RAID array for storing files.

# Step 9 — Create a Mount Point
- Example:
sudo mkdir -p /mnt/storage

- This is where the filesystem will appear.

# Step 10 — Mount the Array
sudo mount /dev/md0 /mnt/storage

- Check:
df -h

- You should see something similar to:
/dev/md0

- mounted at:
/mnt/storage

# Step 11 — Save the RAID Configuration
- Wait until the initial synchronization has completed, then save the array definition:
cat /proc/mdstat

- Once it shows the array is fully synchronized, run:
- Debian/Ubuntu:
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf

- RHEL-based systems:
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf

- This records the array so mdadm can automatically assemble it during boot. Running this after the initial sync helps avoid incorrect array definitions that can lead to unexpected device names (such as /dev/md127) after reboot. 

# Step 12 — Update the Initramfs (Debian/Ubuntu)
sudo update-initramfs -u

- This includes the updated RAID configuration in the early boot environment so the array assembles correctly at startup. 

# Step 13 — Configure Automatic Mounting
- Find the filesystem UUID:
sudo blkid /dev/md0

- Example:
UUID="1234abcd-5678-..."

- Edit /etc/fstab:
sudo nano /etc/fstab

- Add:
UUID=1234abcd-5678-.... /mnt/storage ext4 defaults,nofail 0 0

- Replace the UUID with the one from your system.
- Using the filesystem UUID is generally more robust than relying on a device name like /dev/md0, which can change in some situations.

# Step 14 — Test the Configuration
- Unmount:
sudo umount /mnt/storage

- Remount using fstab:
sudo mount -a

- Verify:
df -h

- No errors means your fstab entry is valid.

# Step 15 — Reboot Test
- Reboot:
sudo reboot

- After logging back in:
- Check RAID status:
cat /proc/mdstat

- View details:
sudo mdadm --detail /dev/md0

- Confirm the filesystem is mounted:
df -h

- Everything should come back automatically.

- Quick Verification Checklist
- Run these commands:
cat /proc/mdstat
sudo mdadm --detail /dev/md0
lsblk
df -h

- A healthy array should:
* show all expected member disks
* report the RAID state as clean or active
* have no failed devices
* be mounted at your chosen mount point
Once these checks pass, your RAID 5 array is ready for use.
