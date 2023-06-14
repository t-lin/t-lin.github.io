---
layout: default
title: Downgrading Ubuntu Desktop to Ubuntu Server
section: wikipost
original_date: 2018-06-22
last_modified: 2018-06-22
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
General steps for safely downgrading Ubuntu Desktop to Ubuntu Server...

```bash
sudo apt-get update
sudo apt-get install tasksel

# Start a second SSH server session running on a separate port (1022)

sudo tasksel remove ubuntu-desktop

# Ensure it completed correctly (sometimes it doesn't, apt-get fails, have to run some other shit)

sudo apt-get install openssh-server # Removing desktop removes this for some reason...
sudo tasksel install server

# Ensure it completes correctly (seems to get stuck on lxd... have to kill process tree, run dpkg --configure -a, apt-get remove lxd and re-run tasksel install server)

sudo apt-get install ubuntu-server # Sanity check to ensure all packages installed

sudo apt-get install linux-generic

sudo apt-get install openvswitch-switch

sudo apt-get autoremove

sudo apt-get purge $(dpkg -l | grep "^rc" | awk '{print $2}') # Clean-up configuration files from old packages

sudo apt-get upgrade # Just in case previous shit didn't fully upgrade

# Edit /etc/default/grub and ensure GRUB_CMDLINE_LINUX_DEFAULT="", then run update-grub

# If configuring an agent server, need to reinstall some shit...
#   - qemu-kvm libvirt-bin lm-sensors landscape-common bmon libmysqlclient-dev

# If moving from 14.04 to 16.04, install upstart again, then reboot, then uninstall upstart after reboot

# Reboot then update + upgrade + autoremove again
```

