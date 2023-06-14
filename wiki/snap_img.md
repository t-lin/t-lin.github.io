---
layout: default
title: Optimizing Snapshot Sizes for Public Use
section: wikipost
original_date: 2019-04-21
last_modified: 2021-01-03
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
**TLIN TODO:** Clean this up

**NOTE:** These notes-to-self were written primarily for OpenStack, but the process of shrinking disk images should generally apply to any situation that calls for it.

```bash
# DOWNSIZE SNAPSHOT IMAGES
#1. download image from glance
#2. connect img as networked block device
#3. mount partition within device
#4. e4defrag on partition (use /dev/... path)
#5. unmount partition
#6. check filesystem (e2fsck) on partition within mounted device
#7. resize filesystem (resize2fs) partition within mounted device
#8. resize (fdisk) partition within mounted device
#8. disconnect networked block device
#9. convert img to raw
#10. shrink raw img via qemu-img resize (repeatedly if need be)
#11. convert raw back to qcow with compression option
#12. upload new img and test

# Connect disk image
glance image-download --file <filename> --progress disk-image-name
qemu-nbd -c /dev/nbd6 path-to-disk-image

# OPTIONAL depending on whether qemu-nbd automatically refreshes partition tables
partprobe /dev/nbd6

# Check fragmentation level on a partition
e2fsck -f -p /dev/nbd6p###

# Mount partition to defrag, if needed
mkdir directory
mount /dev/nbd6p### path-to-directory

# Assuming ext4
e4defrag /dev/nbd6p###

# Optional to zero out non-used areas of partition
mount -o remount,ro {path-to-directory} # Re-mount as read-only
zerofree /dev/nbd6p###

# Check partition data usage, will be used later on
df -h

# Unmount partition
umount path-to-directory


# Downsize filesystem
# Run filesystem check again if defrag was done
# NOTE: resize2fs uses 2^10 notation (i.e. 1024 bytes = 1 kb)
e2fsck -f -p /dev/nbd6p###
resize2fs -p /dev/nbd6p### {target_size}M/G --- OR --- resize2fs -M -p /dev/nbd6p### to shrink to minimum size

# May want to run e2fsck again, sometimes resize2fs changes some inode stuff
# If too much becomes non-contiguous again, may need to repeat defrag (assuming enough free space left)
# CHECK THE SIZE... PARTITION RESIZING (NEXT) MUST NOT BE LESS THAN THIS
e2fsck -f -p /dev/nbd6p###

# Resize partition (delete, re-create using primary, same start location, size to ~100M more than min size)
# CHECK THE END SECTOR OF THE DISK... IMAGE RESIZING (later on) MUST NOT BE LESS THAN THIS
# NOTE: Older versions of fdisk uses powers of 10 notation (i.e. 1000 bytes = 1 KB), verify in manpage
# IF IMAGE IS DISK-ONLY PARTITION, THIS MAY NOT BE NECESSARY (see if partition table exists in fdisk)
fdisk /dev/nbd6

# Disconnect disk image
qemu-nbd -d /dev/nbd6

# Convert image to raw so it can be shrunk
qemu-img convert disk-image-name -O raw disk-image-name.raw

# Check current virtual disk size vs disk size
# Disk size may be reported larger than it actually is, will drop as virtual disk size shrinks
# Repeat as many times as necessary without wiping partition data usage (Leave 100-200 M)
qemu-img info disk-image-name.raw
qemu-img resize disk-image-name.raw +/-{size_change}M/G

# NOTE: Check disklabel type in fdisk
#       If disklabel type is GPT, the secondary header at the end of the disk will have been destroyed
#       by the disk resize operation. Use gdisk to repair it (it'll just take the main table at the
#       start of the disk and duplicate it at the new end-of-disk)
#         - Print table, verify, then write & exit
gdisk /dev/nbd6

# Convert back to qcow and upload
# CHECK TO SEE IF ORIGINAL IMAGE NEEDED RAMDISK + KERNEL, IF SO NEW IMAGE NEEDS THEM AS WELL
# USE COMPAT 0.10 JUST INCASE QEMU NOT UPDATED
qemu-img convert -c disk-image-name.raw -O qcow2 new-image.qcow2
glance image-create --name {name here} --is-public True --container-format bare --disk-format qcow2  < new-image.qcow2
### OR, USING NEW OPENSTACK CLI###
openstack image create --{public/private} --container-format bare --disk-format qcow2 --file new-image.qcow2 {name here}

# Test image ASAP to make sure nothing fucked up

################# Other Notes #################
# If just changing filesystem, can mount via
#   - mount -o loop {image} {directory}
#
# Mounting ramdisk requires specifying specific type:
#   - mount -o loop -t sysfs {image} {directory}
#
# Compatibility w/ qcow2 version 2
#   - qemu-img convert -c {raw image} -O qcow2 -o compat=0.10 {qcow2 image}
#
# Mounting raw disk image with partitions inside
#   - Find offset of partitions first
#       - fdisk -l {disk image}
#   - Then use offset (sector size * start sector)
#       - mount -o loop -o offset={offset in bytes} {image} {directory}
#
# Creating bootable image from a plain filesystem image (no ramdisk or kernel)
#   - First, copy a regular image (with ramdisk and kernel and partition for disk)
#   - Copy the /boot and /run directories from the copied image into the filesystem image
#       - Make sure symbolic links exist in / of filesystem to initrd.img and vmlinuz under /boot
#   - Copy everything from the filesystem image into the desired partition of copied image
#       - Mount them both and use rsync, e.g.: sudo rsync -aHAX --delete --inplace -W {src dir}/ {dest dir}
#           - Do a diff on both directories after to make sure
#   - chroot into copied image, apt-get install the linux-image-`uname -r`
#       - Somehow run sudo depmod afterwards....
#       - Somehow run sudo update-initramfs -u -v -k `uname -r` afterwards....
#           - Check /boot/grub/grub.cfg after and ensure "set root='(hd0,1)'" and "root=LABEL=cloudimg-rootfs" in linux boot line
#   - Ensure "console=ttyS0" is part of GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub?
#   - Ensure GRUB_CMDLINE_LINUX="" in /etc/default/grub to get kernel messages to show when booting
#   - Delete /etc/udev/rules.d/70-persistent-net.rules if in Ubuntu 14.04 or before
#
#until ping -q -w 1 -c 1 8.8.8.8 > /dev/null && echo ok
#do
#    sleep 1
#done
#
# Before chroot'ing into a directory, may need to mount the following:
#   mount --bind /proc /target/proc
#   mount --bind /dev /target/dev
#   mount --bind /sys /target/sys
#
#
# Preparing new images for public consumption (before taking snapshot, or via mounting + chroot)
#   - Delete ~/.cache, bash_history, authorized_keys, and .sudo_as_admin_successful
#   - Remove /etc/hostname file
#   - Remove old instances from /var/lib/cloud/instances
#       - cd /var/lib/cloud/instances && sudo rm -Rf *
#   - Do apt-get clean and remove *verse* files from /var/lib/apt/lists/
#   - Uninstall old linux image and headers
#   - Optional: Set resolvconf.d/head, vimrc, screenrc, tzdata, any other default packages
#   - If daily apt is installed, disable or remove it
#       - Daily apt automatically runs upon VM bootup, may interfere with user-initiated apt tasks
#       - See: https://unix.stackexchange.com/questions/315502/how-to-disable-apt-daily-service-on-ubuntu-cloud-vm-image
#   - If you want to prevent renaming interfaces to ens* or p* from original eth*:
#       - In /etc/default/grub: Make sure "net.ifnames=0 biosdevname=0" is part of GRUB_CMDLINE_LINUX
#       - May need to disable netplan (for Ubuntu 17.10 and above)
#           - Add "netcfg/do_not_use_netplan=true" to GRUB_CMDLINE_LINUX
#           - Remove netplan directory under /etc/ and /lib/
#           - Install "ifupdown" package, then populate /etc/network/interfaces file
#       - Run update-grub after above changes and reboot
#   - If you want to disable systemd-resolvd and use resolvconf:
#       - sudo apt-get install resolvconf
#       - sudo systemctl stop systemd-resolved.service
#       - sudo systemctl disable systemd-resolved.service
#       - Should also disable systemd-networkd so resolvconf can populate using dhcp info
```

