---
layout: default
title: IPMI Serial-Over-LAN (for Supermicro motherboard)
section: wikipost
last_modified: 2017-03-31
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified }}*

# {{ page.title }}
This document describe how to configure and use IPMI serial over lan (SOL). This document created for using with Supermicro motherboard.

## BIOS Configuration
The default BIOS setting on the Supermicro board already have IPMI and serial over lan enabled. The following is the setting that we currently use in BIOS menu:
```
Advanced -> Serial Port Console Redirection -> COM Console Redirection: Enabled
Advanced -> Serial Port Console Redirection -> SOL Console Redirection: Enabled
Advanced -> Super IO Configuration -> SOL Configuration -> SOL serial port: Enabled
Advanced -> Super IO Configuration -> SOL Configuration -> SOL Change Settings: Auto
Advanced -> Super IO Configuration -> SOL Configuration -> SOL Device Mode: Normal
Advanced -> Super IO Configuration -> SOL Configuration -> Serial Port 2 Attribute: SOL
```

## OS configuration - Ubuntu 15.04+
Since Ubuntu 15.04, it has switched to using `systemd` as its main service manager. For Ubuntu versions prior to this, see the next section below.

1. Redirect console to ttyS1 (i.e. Serial Port 2 set in BIOS from previous step)
    1. Create a new file named **/lib/systemd/system/ttyS1.service** and enter the following:

    ```
    [Unit]
    Description=Serial Console Service
    
    [Service]
    ExecStart=/sbin/getty -L 115200 ttyS1 vt100
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    ```

    2. Reload the systemd daemon, enable ttyS1, and start it:

    ```
    sudo systemctl daemon-reload
    sudo systemctl enable ttyS1
    sudo systemctl start ttyS1
    ```
2. Check **/etc/default/grub** and ensure the `GRUB_CMDLINE_LINUX` parameter contains the following arguments: `serial console=ttyS1,19200n8`
    1. Example: `GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0 serial console=ttyS1,19200n8"`
3. Run `sudo update-grub` in terminal

**TROUBLESHOOTING**

If you notice that after server reboots, the serial connection gets flaky (e.g. logging you out after you log in, unable to enter all text properly), this is due to systemctl continuously restarting getty. This appears to be a known bug.

* **Workaround**: In `/etc/rc.local` put a line: `sleep 60 && sudo systemctl daemon-reload && sudo systemctl restart ttyS1 &`

## OS configuration - Ubuntu 12.04 (14.04 should work too)
1. Redirect console to ttyS1 (default serial port)
    1. create a new file called **/etc/init/ttyS1.conf** and type in the following information:

    ```
    start on stopped  rc or RUNLEVEL=[2345]
    stop on runlevel [!2345]
    
    respawn
    exec /sbin/getty -L 115200 ttyS1 vt100
    ```
    2. execute the following: `sudo start ttyS1`
2. Redirect grub (optional, only do this if you need grub interaction through IPMI)
    1. Modify `/etc/default/grub` like the following

    ```
    # If you change this file, run 'update-grub' afterwards to update
    # /boot/grub/grub.cfg.
     
    GRUB_DEFAULT=0
    GRUB_TIMEOUT=1
    GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
    GRUB_CMDLINE_LINUX="console=tty0 console=ttyS1,115200n8"
     
    # Uncomment to disable graphical terminal (grub-pc only)
    GRUB_TERMINAL=serial
    GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
     
    # The resolution used on graphical terminal
    # note that you can use only modes which your graphic card supports via VBE
    # you can see them in real GRUB with the command `vbeinfo'
    #GRUB_GFXMODE=640x480
     
    # Uncomment if you don't want GRUB to pass "root=UUID=xxx" parameter to Linux
    #GRUB_DISABLE_LINUX_UUID=true
    ```
    2. Run `sudo update-grub` in terminal

## Downloading and using the IPMItool client
Command for remote accessing server with IPMI serial over lan configured:

### Windows
1. Download the ipmitool from supermicro’s website:
    * ftp://ftp.supermicro.com/utility/SMCIPMITool/Windows/
2. Run the following command to connect the server using IPMI SOL
    * `.\SMCIPMITool.exe [server address] [username] [password] sol activate`
3. Press F12 to exit

### Ubuntu
1. first install ipmitool by running the following:
    * apt-get install ipmitool
2. Run the following command to connect the server using IPMI SOL
    * `ipmitool -I lanplus -U [username] -P [password] -H [host address] sol activate`
3. Exit by pressing tilde followed by a period: `~.`
