---
layout: default
title: HP 5900 Switch Network OS Commands
section: wikipost
original_date: 2018-11-22
last_modified: 2021-01-31
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
Links to guides:
* <a href="https://techlibrary.hpe.com/device_help/H3C-Manuals/5900/5900-Installation-Guide(20111231).pdf" target="_blank">Installation Guide</a>
* <a href="https://techhub.hpe.com/eginfolib/networking/docs/switches/5920-5900/5200-4534_fund_cg/content/index.htm" target="_blank">Configuration Guide</a>
* <a href="https://techhub.hpe.com/eginfolib/networking/docs/switches/5920-5900/5200-4552_openflow_cg/content/index.htm" target="_blank">OpenFlow Configuration Guide</a>
* <a href="https://techhub.hpe.com/eginfolib/networking/docs/switches/K-KA-KB/15-18/5998_8148_ssw_admin_guide/content/index.html" target="_blank">OpenFlow 1.3 Administration Guide</a>
    * **NOTE:** Front page doesn't list 5900, but the 5900 doesn't have its own administration guide, so this is the closest available one)

High-level seutp notes: https://docs.google.com/document/d/11A2tENiybmnQgxCEFx-PRmQk1Nbc2hVbdmesPKA1yT0/edit

**NOTE:** Remember to save any changes before exiting if you want the configuration to persist after startup.\\
Simply log into the switch and type: `save safely`

## Display status on a port
```
display interface Ten-GigabitEthernet 1/0/<port num>
```

## Change MTU on a port
```
system-view
interface Ten-GigabitEthernet 1/0/<port num>
jumboframe enable <mtu size>
```

## Enabling SSH
```
system-view
local-user <username>
password simple <password>
authorization-attribute user-role network-admin
quit
user-interface vty 0 15
authentication-mode scheme
idle-timeout 0 0
protocol inbound ssh
```

