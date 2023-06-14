---
layout: default
title: Enabling CPU Temperature Monitoring
section: wikipost
original_date: 2019-03-28
last_modified: 2019-10-02
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
**NOTE:** Tested in Ubuntu 16-04 w/ Kernel 4.4.72+

The crucial package to install is `lm-sensors`

The package, however, depends on the `coretemp` kernel module. To include it, install the `linux-image-extras` package for your specific kernel version:\\ `sudo apt-get install linux-image-extra-`uname -r``

Then run:
```bash
sudo sensors-detect
```
* When asked if you want to add lines to either `/etc/modules` or `/etc/sysconfig/lm_sensors`, select **yes**
* If installing from source, there are extra steps:
```bash
sudo cp prog/init/lm_sensors.service /lib/systemd/system
sudo systemctl enable lm_sensors.service
sudo service lm_sensors restart
sudo service lm_sensors status
```

Then restart kmod to load the new modules: `sudo service kmod restart`
* May want to check its status to ensure it successfully restarted (meaning all modules were successfully loaded)
* Can also double-check that coretemp is loaded:
```bash
sudo lsmod | grep coretemp
```
    * If it's not, see if it exists:
    ```bash
ls -l /lib/modules/`uname -r`/kernel/drivers/hwmon | grep coretemp
```
    * If it exists, try loading it:
    ```bash
sudo modprobe coretemp
```

