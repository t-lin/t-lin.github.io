---
layout: default
title: Setting Up Dell EqualLogic PS4210X Storage Array
section: wikipost
original_date: 2018-08-01
last_modified: 2020-06-09
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
This document gives a quick primer on setting up the Dell EqualLogic PS4210X storage array.
Different models may have slightly different instructions, depending on the OS included in the device.

## Initial Setup Requirements
* A serial connection, so a crossover serial cable (female to female) will be needed
* A computer/server with either a serial port or a USB-to-serial adapter

## Setup Steps
1. Connect at least one of the data ports (eth0 or eth1) to a compatible switch
    * **NOTE:** The control modules offer both a 10GE RJ45 and an SFP+ connection port for both eth0 and eth1, so it looks like 4 ports. In functionality, only one or the other (RJ45 or SFP+) will actually be used. If both are connected, the SFP+ will take precedence.
2. Connect the server to the storage array via the crossover serial cable
3. Power on the storage array
4. Start a serial terminal session at 9600 baud
    * If in Linux, can use screen: `sudo screen /dev/ttyS0 9600`
    * In Windows, can use PuTTY or HyperTerminal
5. First time boot-up, the storage array will ask if you want to set up now
    * If you select "no", you can manually start the setup again by typing `setup`
6. Enter a member name (**a "member" is a single storage array server**) and an IP address
    * Note that the IP address should be in the same subnet as the group address (next step)
7. Enter a group name (**a "group" is a set of storage array servers**) and the IP address for the group
    * Note that the IP address should be in the same subnet as the member address (previous step)
    * The server will try to detect the group on the LAN and join it. If the group is not detected, it'll ask if you want to create a new group.
8. At this point you can open a browser and go to the group IP address to launch the Java web GUI to continue configuration of the group and member(s)
    * **NOTE**: The CLI contains certain functions and features that are unavailable via the GUI. The CLI documentation can be downloaded from the Dell support website (login required).

# Integration w/ OpenStack Cinder
The Block Storage service (Cinder) provides block storage devices to guest instances.
The method in which the storage is provisioned and consumed is determined by the Block Storage driver in the single backend mode or drivers in the case of a multi-backend configuration.
There are a variety of drivers that are available: NAS/SAN, NFS, iSCSI, Ceph, and more.
The following tasks should be followed to set up Cinder:
* Edit `cinder.conf` based on the following instructions: <a href="https://docs.openstack.org/cinder/latest/install/cinder-storage-install-ubuntu.html" target="_blank">Install and configure a storage node</a> and <a href="https://docs.openstack.org/liberty/config-reference/content/dell-equallogic-driver.html" target="_blank">Dell EqualLogic volume driver</a>
* Install and configure iSCSI on the Cinder server as an iSCSI initiator  `sudo apt install -y open-iscsi`
    * Edit `/etc/iscsi/iscsid.conf`, and ensure the `node.startup` field is `automatic`
    * On the EqualLogic, enable iSCSI conncections (may require use of the CLI)
    * Finally, by issuing `sudo iscsiadm -m discovery` from the Cinder server, the target iSCSI (i.e. the EqualLogic Storage server) and its iSCSI Qualified Name (IQN) should be accessible
        * e.g. `10.20.30.40:3260,iqn.1992-05.com.equal:sl7b92030000520000-2`

## Troubleshooting
* If the EqualLogic server is not found by automatic discovery, try manually providing the IP, e.g. `sudo iscsiadm -m 
  discovery -t st -p <EqualLogic IP>` 
* If Cinder cannot connect to the storage server, check the log files under `/var/log/cinder/*.log` (or wherever it was configured to store logs)
* Further troubleshooting: <a href="https://docs.openstack.org/cinder/latest/admin/ts-cinder-config.html">Troubleshoot the Block Storage configuration</a>

# Manually Creating & Attaching Virtual Volumes
The following steps enables us to manually create a virtual volume on the storage server, and connect it to a server of our choice (VM or baremetal).
**NOTE**: This procedure assumes the server(s) in question have network connectivity to the storage server.
1. See current list of devices in the server: `ls -l /dev/sd*` 
2. Connect to the storage server's GUI and log in
    * If the server is in a separate network, use dynamic port forwarding to access it, and configure Java to use SOCKS proxy
3. Create a volume as desired in the GUI
4. In the server, test to see if the volume is discoverable:
    * `sudo iscsiadm --mode discoverydb --type sendtargets --portal 10.20.40.200 --discover`
    * The above command should list available volume targets
5. If discoverable, then try to connect to the new volume:
    *  `sudo iscsiadm --mode node --targetname <target name from step 4> --portal 10.20.40.200 --login`
6. If the connection was successful, repeat step 1, and see if a new device has appeared

{% comment %}
# Additional Resources
  * Dell EqualLogic PS4210 Storage Arrays: Installation & Setup Guide
  * Dell EqualLogic PS4210 Storage Arrays: Hardware Owner's Manual
  * Dell PS Series Configuration Guide
  * Dell EMC Networking S4048-ON Switch Configuration Guide for PS Series
{% endcomment %}

