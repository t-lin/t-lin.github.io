---
layout: default
title: Installing or Upgrading PicOS OVS
section: wikipost
original_date: 2018-01-07
last_modified: 2021-02-18
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified }}*

# {{ page.title }}
This guide shows how to install or upgrade PicOS from scratch.

## Connecting to Serial Console
Default baud rate: 115200
8 data bits, no parity, 1 stop bit

## Getting Into Flash Linux Environment
Bypass the default U-Boot menu and get into the flash Linux environment.
  - Attach to the serial console
  - Reboot the server
  - Press any key when you see the `Hit any key to stop autoboot` message
  - Type: `run flash_bootcmd`
  - When you see the Diagnostic Test Main Menu, hit <kbd>Ctrl+D</kbd> twice

From here you now have a basic Linux environment. You can check `dmesg` logs for any errors and etc. The PicOS firmware should be stored in a directory named `/cf_card`

## First-Time Installation or Re-Building the System
First get into the flash Linux environment (see above). If `/cf_card` already exists with a previous firmware installed, you can skip to the next section about upgrading.

Otherwise, you'll need to configure the compact flash and set this directory up yourself as follows:
1. See if the compact flash is mounted via either `df` or `mount`
    * Check if /dev/hda is mounted
2. Unmount the compact flash card if needed (e.g. `umount /cf_card`)
3. Partition the card, the first partition is where we will place the PicOS system:
    1. `fdisk /dev/hda`
    2. Delete the first partition if it exists, and re-create it if needed, otherwise just create
    3. Remember to write your partition changes to the disk before exiting
4. Create the filesystem: `mke2fs -j /dev/hda1`
5. Mount the new filesystem: `mount /dev/hda1 /cf_card`

From here, you can continue on to the section about upgrading (the steps should be identical).

## Upgrading the PicOS System
This step involves fetching a new image from another (preferably local) computer via TFTP. The other server should have a TFTP server running and the new PicOS image (in a .tar.gz format) already sitting in its upload directory.
  * Can use `tftpd-hpa` package for the server
  * The default upload directory should be at `/var/lib/tftpboot`
  * Default port should be 69
  * These settings can be changed via the configuration file at: `/etc/default/tftpd-hpa`

To upgrade PicOS, first get into the flash Linux environment (see above). The PicOS installation should be in the `/cf_card` directory.
1. `cd` to the `/cf_card` directory and ensure it's empty (it's fine for `lost+found` directory to be there)
2. Assign the management interface of the switch (should be `eth0` within the switch) an IP address
3. Make sure the management interface is connected to other server running the TFTP server
4. Retrieve the image from the server: `tftp -g -l rootfs.tar.gz -r <remote_filename> <remote_server_ip>`
    * This may take several minutes
5. Untar the image: `tar zxvf rootfs.tar.gz`
    * **Don't** delete the file afterwards, it's used for checking the firmware checksum during booting
6. Write the changes back to the flash: `sync`
7. Reboot the switch: `reboot`
    * This time, don't interrupt the autoboot when asked

**NOTE:** The first time rebooting with a new firmware will take several minutes

## Switching PicOS from XorPlus to OVS Mode (One-Time Change)
By default, PicOS will boot into it's "XorPlus" mode. This can be modified to boot into a pure OVS mode.
1. Log into the OS. The default username/password should be: `admin` / `pica8`
2. Run `picos_boot` and change to PicOS OVS/OpenFlow mode, entering whatever configuration questions asked
3. Reboot the switch one more time: `reboot now`

**NOTE:** The first time rebooting into PicOS OVS mode may take several minutes

## Setting Up and Configuring OVS
From within the new PicOS OVS environment you can do the following:
  * Change root password if desired
  * Interact with the switch using `ovs-vsctl` and `ovs-ofctl` commands

To create an OVS bridge and add the switch's interfaces, you need to nake sure the bridge is created as type `pica8` and add the interfaces as type `pica8` as well. For example:
```
ovs-vsctl add-br br0 -- set bridge br0 datapath_type=pica8
ovs-vsctl add-port br0 te-1/1/1 -- set interface te-1/1/1 type=pica8
```

Examples to configure the DPID, the OpenFlow version, or set any interfaces to 1 gigabit mode:
```
ovs-vsctl set bridge br0 protocols=OpenFlow10
ovs-vsctl set bridge br0 other-config:datapath-id=0000010010030093 # Has to be 16 chars
ovs-vsctl set-controller br0 tcp:10.10.60.10:6633
ovs-vsctl set interface te-1/1/11 options:link_speed=1G
```

## Helpful Links
http://www.pica8.com/wp-content/uploads/2015/09/v2.0/pdf/2.0.1-image-upgrade-guide.pdf
http://www.pica8.com/documents/html/ovs_configuration_guide/1639927.html

## Troubleshooting
Potential causes for various issues, and how to fix them.

### Not auto-booting into switch OS
  * Follow steps 1-3 of the **Getting Into Flash Linux Environment** section above
  * Type `printenv`, and check what the `bootcmd` variable is, or what other variable it runs

The following is a sample environment from after upgrading a 10 GE switch from Indigo to PicOS. Note that `bootcmd` runs `cfcard_bootcmd2`, which is for PicOS. There's also an older `cfcard_bootcmd` which was for Indigo. You can change the `bootcmd` variable to run any other variable, just remember to **save** the new configuration to the on-board flash.
```
=> printenv
flash_bootcmd=setenv bootargs root=/dev/ram console=ttyS0,$baudrate; bootm ffd00000 ff000000 ffee0000
cfcard_bootcmd=setenv bootargs root=/dev/ram console=ttyS0,$baudrate; ext2load ide 0:1 0x1000000 /uImage;ext2load ide 0:1 0x2000000 /uInitrd2m;ext2load ide 0:1 0x400000 /LB9A.dtb;bootm 1000000 2000000 400000
bootdelay=5
baudrate=115200
loads_echo=1
ipaddr=192.168.2.1
serverip=192.168.2.12
rootpath=/nfsroot
gatewayip=192.168.2.254
netmask=255.255.255.0
hostname=LB9A_X
bootfile=eldk-quanta
loadaddr=4000000
ethact=TSEC0
filesize=1CD2AB
cfcard_bootcmd2=setenv bootargs root=/dev/hda1 rw noinitrd console=ttyS0,$baudrate; ext2load ide 0:1 0x1000000 boot/uImage;ext2load ide 0:1 0x400000 boot/LB9A.dtb;bootm 1000000 - 400000
bootcmd=run cfcard_bootcmd2
ethaddr=e8:9a:8f:91:af:97
eth1addr=e8:9a:8f:91:af:98
stdin=serial
stdout=serial
stderr=serial
```

