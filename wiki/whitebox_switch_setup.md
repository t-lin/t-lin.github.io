---
layout: default
title: Setting Up EdgeCore Wedge Switches
section: wikipost
original_date: 2021-08-05
last_modified: 2021-08-10
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
Setting up a whitebox switches often involves installing various firmware on it and configuring the on-board network operating system (NOS).

**Note**: This document will focus on high-level steps for setting up the **EdgeCore Wedge 100BF-32X**. The switch comes with two on-board computer systems: a "mainboard" system and a Computer-on-Module (COM) Express (COMe). The mainboard computer contains a lightweight and super-thin OS that can be used to interface with the COMe and install one or more NOS into the COMe. This document will focus on the installation of a NOS into the COMe, and ignore the mainboard OS.

The mainboard OS that comes pre-loaded into the EdgeCore Wedge switch is a customized OpenBMC OS (i.e. it contains numerous EdgeCore/Wedge scripts) with a default username and password: root/0penBmc

## Attaching to & Interacting w/ COMe
By default, the switch's serial cable allows access to the mainboard system. There is a serial-over-LAN (SOL) interface between the mainboard and the COMe system that can be used to attach to the COMe and interact with it.

The mainboard OS may also have acquired a local IP address on the LAN (if there's a local DHCP server running) that can allow SSH access. There's currently no way to set its default IP address since any changes to the mainboard OS are wiped upon reboot.
* For the current Wedge switch, its mainboard's interface MAC address is **F8:8E:A1:71:3A:98**; a simple scan of the LAN can be used to discover which IP address the mainboard obtained.

### Toggling the COMe's Power
From the mainboard OS, the COMe's power status can be queried, or it can be turned on/off or even reset. From the mainboard OpenBMC OS, there is a script called `wedge_power.sh` that facilitates this process:
```bash
root@bmc:~# wedge_power.sh -h
Usage: /usr/local/bin/wedge_power.sh <command> [command options]

Commands:
  status: Get the current microserver power status

  on: Power on microserver if not powered on already
    options:
      -f: Re-do power on sequence no matter if microserver has
          been powered on or not.

  off: Power off microserver ungracefully

  reset: Power reset microserver ungracefully
    options:
      -s: Power reset whole wedge system ungracefully
```

### Attaching to / Detaching from the COMe
To invoke the SOL interface and attach to the COMe, there is a `sol.sh` script that facilitates this procedure:
```bash
root@bmc:~# sol.sh
You are in SOL session.
Use ctrl-x to quit.
-----------------------


sonic login:
```

**NOTE**: You may need to press enter at least once to see anything. In the above example, we see `sonic login:` because we have installed the SONiC OS into the COMe.

If nothing is seen, the COMe may be off and needs to be turned on (refer to previous section). Worst case scenario, the COMe may need to power cycled (i.e. off then on again).

To detach from the COMe (i.e. exit the SOL) and return to the mainboard system, simply press **ctrl+x**.

### Getting into BIOS of COMe
This part is straight-forward; simply reset the COMe and attach to its SOL interface quickly.
```bash
root@bmc:~# wedge_power.sh reset; sol.sh
```

You'll see the regular boot-up procedure of servers (i.e. a message early in the boot sequence about pressing the "Delete" key). You can customize the functionality of the COMe system from there.

## Building & Installing ONIE into the COMe
The Open Network Install Environment (ONIE) system is a thin Linux OS that can be installed into the COMe to facilitate the installation of other NOS onto the same system (it essentially simplifies NOS installation, upgrade, and uninstallation).

The Wedge should have come with ONIE already installed into the COMe. If not, it can be built as follows:
* Create a build environment in using Dedicated User Environment (DUE)
```bash
git clone https://github.com/CumulusNetworks/DUE.git && cd DUE
./due --create --from debian:9  --description "ONIE Build Debian 9" --name onie-build-debian-9 --prompt ONIE-9 --tag onie --use-template onie
```
    * The above creates a build environment in a container using Debian 9 as a base
* Run build environment:
```bash
./due --run
```
    * The above should default to the ONIE build container you just created if it's the only container image made by DUE; If there's more containers, you may have to specify which one to run
    * Your home directory is mounted into the container in the same path, so any code/files in your home should also be available in the container (this is useful for pre-populating the resulting ONIE image)
        * You can tell you're you're in the container is via the hostname
* From within the build environment, clone & build ONIE
    * `git clone https://github.com/opencomputeproject/onie.git && cd onie/build-config`
    * There's a list of potential build targets w/ different architectures in `scripts/onie-build-targets.json` (see the `Platform` column)
      * Example: If building for the Wedge 100BF-32X-O-AC-F-US, the closest is `accton_wedge100bf_32x`
    * Kick off the build while invoking the target architecture, e.g.:
      * `make MACHINEROOT=../machine/accton MACHINE=accton_wedge100bf_32x all`
    * The resulting files will be in the images subdirectory relative to the root onie directory, e.g.
    ```bash
savi@savi-iam-v3:~/wedge-setup/onie/build-config$ ls -l ../build/images/
total 82744
-rw-r--r-- 1 savi savi  8655072 Jul 25 03:34 accton_wedge100bf_32x-r0.initrd
-rw-r--r-- 1 savi savi  3723456 Jul 25 03:24 accton_wedge100bf_32x-r0.vmlinuz
-rw-r--r-- 1 savi savi 31268864 Jul 25 03:38 onie-recovery-x86_64-accton_wedge100bf_32x-r0.efi64.pxe
-rw-r--r-- 1 savi savi 28639232 Jul 25 03:38 onie-recovery-x86_64-accton_wedge100bf_32x-r0.iso
-rw-r--r-- 1 savi savi 12433271 Jul 25 03:34 onie-updater-x86_64-accton_wedge100bf_32x-r0
```
    * The \*.iso file can be used to install/reinstall ONIE (may require physical access to the switch)
        * See: https://support.edge-core.com/hc/en-us/articles/360021708534--ONIE-How-to-recover-ONIE-on-Edgecore-x86-platform-switch-via-USB

## Booting into ONIE
Once ONIE is installed, you should be able to boot it by simply turning on (or resetting) the COMe and attaching to the COMe using SOL. If the ONIE installation went well, you should see a grub menu. It's important to quickly interrupt the grub from booting into its default option and instead go into the **ONIE Rescue** mode.

**NOTE**: Since ONIE is a thin OS meant for installing other NOSes, its default grub option will be to boot into "installation" mode, where it continuously scans hosts on the LAN for a server to download a NOS image from. At the time of writing, there's no way to exit the continuous scan other than re-setting the COMe from the mainboard OS.

ONIE rescue mode will acquire an IP address via DHCP over the LAN (separate from the mainboard OS's interface). This can allow simpler access to the system.
* For the current Wedge switch, its COMe interface MAC address is **00:90:FB:70:65:5D**; a simple scan of the LAN can be used to discover which IP address the COMe obtained.

### Making ONIE Rescue Mode Default
ONIE has a flag to help determine whether a NOS has been installed (in which case, its grub won't automatically go into installation mode). To set this flag, boot into ONIE rescue mode and run:
```bash
ONIE:/ # onie-nos-mode -s # Sets ONIE rescue as default grub option
ONIE:/ # onie-nos-mode -g # Queries current NOS mode flag
```

**NOTE**: There's another important reason to set this flag. When a new NOS is installed, the default grub on the disk (i.e. installed on /dev/sda) is modified to boot from the new NOS (a secondary grub option will exist to allow access to ONIE's grub menu). If installation mode is accidentally allowed to run, it wipes the default grub on disk and makes it always boot into ONIE's grub menu). This will be a bit tricky to fix.

### Installing a NOS
From the ONIE rescue mode, a new NOS can be installed over the network. By default, ONIE expects a local host to be able to serve the NOS image (e.g. Open Network Linux (ONL), SONiC, etc.) over HTTP, so a local web server must be set up. The image can then be fetched by ONIE via the `onie-nos-install` script, e.g.:
```bash
ONIE:/ # onie-nos-install http://10.23.100.22:8080/Edgecore-SONiC_20201229_07031
```

The NOS installation will take a while. After its complete, it will re-write the default grub on the disk (e.g. /dev/sda) to set the NOS image as the default option so future boots should automatically go into the NOS instead of ONIE. If the installation process doesn't automatically reboot, then manually reboot the COMe system.

**NOTE**: For the SONiC NOS, the default username/password is: admin/YourPaSsWoRd

## Configuring Switch with SONiC
These are a few high-level configuration options. The main command reference for SONiC can be found here: https://github.com/Azure/sonic-utilities/blob/master/doc/Command-Reference.md

**SAVING CONFIGURATION CHANGES**: Remember to save the current configuration to disk to ensure the changes you make persist across power outages. This is achieved as follows:
```bash
admin@sonic:/etc/sonic$ sudo config save
```

The configuration will be stored into `/etc/sonic/config_db.json`, which is where the boot-up configuration for the switch is saved.

### Querying Port Status
**NOTE**: By default, SONiC names each port as `Ethernet0`, `Ethernet4`, `Ethernet8`, ... to `Ethernet124`. This is because each 100G port can be broken up into 4x10G or 4x25G in break-out mode.
  * The translation from the front-panel number to the SONiC-specific number is straight-forward: `(<panel #> - 1) * 4`

The port status can be queried via the `show` script:
```bash
admin@sonic:~$ show interfaces status
  Interface            Lanes    Speed    MTU    FEC    Alias    Vlan    Oper    Admin            Type    Asym PFC
-----------  ---------------  -------  -----  -----  -------  ------  ------  -------  --------------  ----------
  Ethernet0                0      10G   9100   none   Eth1/1   trunk      up       up  QSFP+ or later         N/A
  Ethernet1                1      10G   9100   none   Eth1/2   trunk      up       up             N/A         N/A
  Ethernet2                2      10G   9100   none   Eth1/3  routed    down     down             N/A         N/A
  Ethernet3                3      10G   9100   none   Eth1/4  routed    down     down             N/A         N/A
  Ethernet4          4,5,6,7     100G   9100   none   Eth2/1  routed    down       up             N/A         N/A
  Ethernet8        8,9,10,11     100G   9100   none   Eth3/1  routed    down       up             N/A         N/A
 Ethernet12      12,13,14,15     100G   9100   none   Eth4/1  routed    down       up             N/A         N/A
 Ethernet16      16,17,18,19     100G   9100   none   Eth5/1  routed    down       up             N/A         N/A
 Ethernet20      20,21,22,23     100G   9100   none   Eth6/1  routed    down       up             N/A         N/A
...
```

In the example above, port 1 of the switch has been broken out into 4x10G, and so we see `Ethernet0` to `Ethernet3`. Note that only the original interface will show the interface type (e.g. `Ethernet0` shows type `QSFP+`, but `Ethernet1` to `Ethernet3` shows `N/A`).

### Configuring Breakout Ports
Breakout ports can be configured via the `config` script.
```bash
admin@sonic:~$ sudo config interface breakout <interface name> <mode>
```

Possible `<mode>` options are:
  * 1x100G[40G]
  * 2x50G
  * 4x25G
  * 4x10G

**NOTE**: To undo the breakout, simply set the breakout back to `1x100G[40G]` (the `[40G]` must be specified)

### Enabling Simple L2 Switching
To enable simple L2 switching, a VLAN must be created and ports must be added to it in untagged mode.
```bash
admin@sonic:~$ sudo config vlan add <vlan #>
admin@sonic:~$ sudo config vlan member add -u <vlan #> <interface name> # the '-u' means untagged
```

The VLAN configurations can then be reviewed using either of the following commands.
```bash
admin@sonic:~$ show vlan brief
admin@sonic:~$ show vlan config
```

### Toggling Ports
Individual ports can be toggled (enabled/disabled) via:
```bash
sudo config interface shutdown <interface name>
sudo config interface startup <interface name>
```

