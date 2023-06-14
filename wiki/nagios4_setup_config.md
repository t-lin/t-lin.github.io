---
layout: default
title: Nagios 4 Setup & Configuration
section: wikipost
original_date: 2018-07-30
last_modified: 2019-07-24
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
Since Nagios 4 is still unavailable in Debian/Ubuntu's upstream apt repos, this article will present a high-level guide on how to manually set up Nagios 4 on Ubuntu 16.04, as well as how to customize/configure it.

## Pre-Req Packages
The latest packages should be installed via apt-get:
```
apache2 libapache2-mod-php7.0 php7.0 wget unzip zip autoconf gcc libc6 make apache2
```

### Troubleshooting Apache2 Startup
If you have Apache2 running within containers, you may have trouble starting Apache2 in the main host. When trying to start Apache2, you may encounter an error with this error message: *"There are processes named 'apache2' running which do not match your pid file ..."*

To get around it, edit **/etc/init.d/apache2**
- Find lines with: `pidof $DAEMON`, and replace with: `pgrep --ns 1 --nslist uts $NAME`
- Reload systemctl: `sudo systemctl daemon-reload`
- Try starting Apache2 again

## Installation of Nagios Core
1. Get the latest stable release code from: https://www.nagios.org/downloads/nagios-core/
2. Untar it and `cd` into the resulting directory
3. Run the configuration script: `./configure --with-httpd-conf=/etc/apache2/conf-available`
4. Compile Nagios: `make all`
5. Copy the resulting binaries, set up the init scripts, create the users, and etc.
    - ```
sudo make install-groups-users
sudo make install
sudo make install-init
sudo make install-daemoninit
sudo make install-config
sudo make install-commandmode
sudo make install-webconf
```
6. Install Nagios plugins (this should also be done in the agents later)
    1. Get the latest plugin code from: https://www.nagios.org/downloads/nagios-plugins/
    2. Untar it and `cd` into the resulting directory
    3. Run the configuration script: `./configure`
    4. Compile the plugins: `make`
    5. Copy the resulting binaries: `sudo make install`
7. Create an admin account named nagiosadmin: `htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin`
8. Add the 'nagios' user to the 'www-data' group: `sudo usermod -aG nagios www-data`

The resulting installation from the above steps should place all the relevant Nagios files under **/usr/local/nagios**

You can see the systemd service script uses this directory for the binary, configuration scripts, and log files. The service script should be at **/lib/systemd/system/nagios.service**

The plugins can be found in **/usr/local/nagios/libexec**

## Enabling Nagios in Apache
The installation should have created a **nagios.conf** file within **/etc/apache2/conf-available**

1. If Nagios Core isn't currently running, start it: `sudo service nagios restart`
2. Enable the `cgi` module in Apache: `sudo a2enmod cgi`
3. To enable the web GUI in Apache (if not already), run: `sudo a2enconf nagios`
4. After the above commands, restart Apache: `sudo service apache2 restart`

### Changing Default Apache Port
By default Apache will bind to and listen on TCP port 80. To change this, edit **/etc/apache2/ports.conf** and restart Apache afterwards.

### Verify Nagios Works w/ Apache
Open up a browser and go to: **http://\<SERVER IP\>:\<APACHE PORT\>/nagios**


## Installing Nagios Remote Plugin Executor (NRPE)
The NRPE plugin allows Nagios to monitor remote hosts via a thin agent executor installed within each host.

**Note:** Nagios plugins will have to be installed in the remote hosts as well (hence the name "remote plugin executor").

First, we will install the NRPE plugin into Nagios Core (i.e. the same server where Nagios Core was installed).
1. Download the NRPE plugin from: https://www.nagios.org/downloads/nagios-core-addons/
2. Untar it and `cd` into the resulting directory
3. Run the configuration script: `./configure`
4. Compile the plugin: `make all`
5. Install the plugin: `sudo make install-plugin`
6. Edit the file **/usr/local/nagios/etc/nagios.cfg**
    * Uncomment the line `cfg_dir=/usr/local/nagios/etc/servers`
    * Save & exit
    * Make the above directory if it doesn't already exist: `sudo mkdir -p /usr/local/nagios/etc/servers`
      * This will be used later to store configuration files for each remote host Nagios is monitoring
7. Edit the file **/usr/local/nagios/etc/objects/contacts.cfg**
    * Enter an email name to send alerts to (see the big all-caps comment message)
    * Save & exit
8. Edit the file **/usr/local/nagios/etc/objects/commands.cfg**
    * Add the new command below, then save & exit:
    ```
define command{
    command_name check_nrpe
    command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
```

### Adding Remote Hosts
To add remote hosts to be monitored, it needs to install the NRPE daemon and Nagios plugins.
1. Install the pre-req packages in the agents: `sudo apt-get -y install gcc openssl libssl-dev make`
2. Install the Nagios plugins (instructions in a previous section above)
3. Download or copy the existing NRPE plugin code
4. Run the configuration script: `./configure`
5. Compile the plugin: `make all`
6. Create the user and groups: `sudo make install-groups-users`
7. Install the daemon: `sudo make install-daemon`
8. Install the configuration file: `sudo make install-config`
9. Edit the configuration file **/usr/local/nagios/etc/nrpe.cfg**
    * Find the `allowed_hosts` line and append the controller's IP address to the end (don't forget the comma separator)
    * Find and modify the `check_hda1` command
      * Change the command name to `check_disk` and ensure the device name at the end matches the root disk
      * **Bash script to do the above is shown below**
```
sudo sed -i "s/check_hda1/check_disk/g" /usr/local/nagios/etc/nrpe.cfg
MAIN_DISK=`df -h / | tail -n1 | awk '{print $1}'`
sudo sed -i "s/\/dev\/hda1/"${MAIN_DISK//\//\\/}"/" /usr/local/nagios/etc/nrpe.cfg
```
10. Install the systemd init file and enable the service:
```
sudo make install-init
sudo systemctl enable nrpe
sudo service nrpe restart && sudo service nrpe status
```

#### Steps In Nagios Core Server
The following steps should be completed in the server where Nagios Core is installed.
1. Verify that Nagios Core can contact the remote executor
    * Go to the plugin directory (**/usr/local/nagios/libexec**) and execute: `./check_nrpe -H <AGENT HOSTNAME>`
2. Add a config file for the agent in **/usr/local/nagios/etc/servers**, the filename should be `<AGENT HOSTNAME>.cfg`
    * Use the template file below, replace `AGENT_NAME` with the agent's hostname
    * Template config file:

    ```
    define host {
        use                             linux-server
        host_name                       AGENT_NAME
        alias                           AGENT_NAME
        address                         AGENT_NAME
        max_check_attempts              5
        check_period                    24x7
        notification_interval           30
        notification_period             24x7
    }
    
    define service {
        use                             generic-service
        host_name                       AGENT_NAME
        service_description             CPU load
        check_command                   check_nrpe!check_load
    }
    
    define service {
        use                             generic-service
        host_name                       AGENT_NAME
        service_description             Disk free space
        check_command                   check_nrpe!check_disk
    }
    ```
    * In the configuration above, the command after the exclamation mark (i.e. `'!'`) is the command that will be executed in the remote hosts (e.g. `check_load`, `check_disk`)
      * To see what this actually runs in the remote host, go to the agent and run: `cat /usr/local/nagios/etc/nrpe.cfg | grep '^command`'

