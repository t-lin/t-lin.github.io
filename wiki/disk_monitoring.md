---
layout: default
title: Setting Up Disk Monitoring
section: wikipost
original_date: 2018-07-20
last_modified: 2018-07-20
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
Disk monitoring can be done via the package `smartmontools`

To get e-mail alerts, the server should have a mail transport agent (MTA) installed. Suggest to install `postfix`.
- When installing `postfix`, select Internet Site, and specify the hostname of the server for identification purposes
- Suggested configuration of the server (edit **/etc/postfix/main.cf**):
    - Make server listen only on loopback interface: `inet_interfaces = loopback-only`
    - Set hostname of server: `myhostname = <HOSTNAME HERE>`
- Test sending an email, e.g.: `mail -s "test subject" <EMAIL HERE> < file-with-test-body-content.txt`

Set the e-mail address for sending alerts:
- Edit **/etc/smartd.conf**, comment out the original `DEVICESCAN` line
- Replace it with: `DEVICESCAN -H -l error -l selftest -n standby -m <EMAIL HERE> -M exec /usr/share/smartmontools/smartd-runner`
- If server includes HDDs that can't be detected (some removable HDDs are like this), manually specify device **before** the `DEVICESCAN` line
    - Example: `/dev/sdc -d sat -d removable -H -l error -l selftest -n standby -m <EMAIL HERE> -M exec /usr/share/smartmontools/smartd-runner`
    - Adjust the type(s) after `-d` as required

