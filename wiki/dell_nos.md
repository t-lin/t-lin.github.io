---
layout: default
title: Dell Networking OS Commands
section: wikipost
original_date: 2017-03-08
last_modified: 2020-03-02
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
This page contains commands on setting up, configuring, and debugging a Dell switch running Dell Networking OS firmware.

The commands here have been tested with the Dell S3048-ON (48x1GE) and S4048-ON (48x10GE) switches. Full configuration manual for the S3048 can be found online.

**Note:** First time configuration of switch requires connecting serial to the console port \\
From Ubuntu, can access serial using `screen`, e.g.:
```
sudo screen /dev/ttyS0 115200
```

**Note:** When entering **enable** mode, you will need to provide a password (configured during first setup).

----
# Configuring Dell Networking OS Switch
This section contains commands on setting up and configuring a Dell switch running Dell Networking OS firmware.

**SAVING CONFIGURATION CHANGES:** Remember to copy the current configuration to the flash to ensure the changes you make persist across power outages. This is achieved as follows:
```
enable
copy running-config startup-config
```

## Configuring management interface IP address
Requires `CONFIGURATION` mode.
```
enable
configure
interface managementethernet 1/1
ip address <ip address>/<cidr>
no shutdown
```

## Configuring host name
Requires `CONFIGURATION` mode.
```
enable
configure
hostname <host name>
```

## Enabling Remote Administration
Remote administration requires SSH access and the `enable` password to gain privileged access once SSH'd in. It's assumed the following steps will be done over serial console access.

### Enabling SSH server
Requires `CONFIGURATION` mode.
```
enable
configure
ip ssh server enable
```

### Create an account
Requires `CONFIGURATION` mode. If continuing from the previous section, can skip the `enable` and `configure` commands below.
```
enable
configure
username <username> password 0 <password>
```

### Create `enable` secret
Requires `CONFIGURATION` mode. If continuing from the previous section, can skip the `enable` and `configure` commands below.
```
enable
configure
enable password 0 <password>
```

## Enabling Simple L2 Switching
Requires `INTERFACE` mode.
```
enable
configure
interface gigabitethernet 1/<port number>
no shutdown
switchport
```

### Configuring a range of interfaces
Replace `interface gigabitethernet 1/<port number>` above with:
```
interface range gigabitethernet 1/<start port number>-1/<end port number>
```

### Checking which ports in switching mode
Requires `EXEC` mode (i.e. `enable`)

```
enable
show interfaces switchport
```

## Creating Port-Based VLANs
By default the switch comes with a single VLAN with ID 1. The previous section on enabling L2 switcing, where we set ports to `switchport`, was essentially adding them into VLAN 1.

### Listing VLANs
Requires `EXEC` mode (i.e. `enable`)

```
enable
show vlan
```

### Creating Port-Based VLANs
Requires `CONFIGURATION` mode
Now we want to create a different VLAN and add ports to that.

```
enable
configure
interface vlan <new VLAN ID>
```

### Adding Ports to VLANs
Requires `INTERFACE VLAN` mode

**Note:** Ports can only be added to VLANs if they're already enabled for L2 switching (i.e. `switchport` was set)

```
enable
configure
interface vlan <VLAN ID>
untagged gigabitethernet 1/<port number>
```

Can also specify a range of interfaces, e.g. for ports 45 to 48:
```
untagged gigabitethernet 1/45-1/48
```

### Removing Ports from VLANs
Removing ports is similar to adding ports, just add a `no` to the beginning of the command; e.g.:

```
no untagged gigabitethernet 1/<port number>
```

## Creating Tagged VLANs
The procedure to create tagged is very similar to the port-based VLANs. If you know the VLAN ID you need, create a VLAN with that ID.

### Creating Port-Based VLANs
Requires `CONFIGURATION` mode
Now we want to create a different VLAN and add ports to that.

```
enable
configure
interface vlan <new VLAN ID>
```

### Configuring Trunked VLAN Ports
Trunk ports allow inter-switch tagged VLAN links. Configuring the interface to be a trunk port requires `INTERFACE` mode.

```
enable
configure
interface gigabitethernet 1/<port number>
no shutdown
no switchport
switchport mode private-vlan trunk
```

### Adding Ports to VLANs
Ports can be added to the VLAN in untagged or tagged mode. Tagged ports will tag the packets as they exit the switch, and untag the packets as they come in. Requires `INTERFACE VLAN` mode.

**Note:** Ports can only be added to VLANs if they're already enabled for L2 switching (i.e. `switchport` was set)

The following snippet adds a port in tagged mode.
```
enable
configure
interface vlan <VLAN ID>
tagged gigabitethernet 1/<port number>
```

## Configuring OpenFlow
Enabling OpenFlow on the switch requires re-configuring and re-allocating the ACL CAM. The ACL CAM settings have odd requirements, namely:
  - Total blocks allocated must equal 13
  - Can only have one ACL with block size 1

For more information, see the "Dell OpenFlow Deployment and User Guide 4.0" guide which can be found online.

### Suggested CAM Setup for S4048-ON
Requires `CONFIGURATION` mode
```
enable
configure
cam-acl l2acl 2 ipv4acl 2 ipv6acl 0 ipv4qos 0 l2qos 1 l2pt 0 ipmacacl 0 vman-qos 0 ecfmacl 0 openflow 8
cam-acl-vlan vlanopenflow 1 vlaniscsi 1 vlanaclopt 0
```

After applying the above, `reload` the switch.

### Configuring an OpenFlow Instance
Requires `CONFIGURATION` mode. Switch supports creating up to 8 OpenFlow instances.\\
**Note:** Re-configuring an OpenFlow instance can only be done when it is `shutdown`

Example configuration to:
  - Create an OpenFlow instance with ID 1
  - Set controller to a given TCP IP & port
  - Set OpenFlow version to 1.0
  - Set the instance's datapath ID (DPID)

```
enable
configure
openflow of-instance 1
controller 1 <ip address> port <port number> tcp
of-version 1.0
dpid-mac-addr <48-bit colon-separated DPID>
```

The DPID above will automatically be pre-pended by 00:01 to satisfy OpenFlow's 16-hex digit requirement.

### Adding Ports to an OpenFlow Instance
Requires `INTERFACE` mode. Interface **must not be in L2 mode** (i.e. run `no switchport` if need be).
Make sure the OpenFlow Instance is disabled. Just follow the steps below for "Enabling an OpenFlow Instance" except do: **shutdown**
```
enable
configure
interface range tengigabitethernet 1/<start port number>-1/<end port number>
no switchport
no shutdown
of-instance <OpenFlow instance ID>
```

Note: If you just want to add one port instead of a range, use the following command:
```
interface tengigabitethernet 1/<port-number>
```

### Enabling an OpenFlow Instance
Requires `CONFIGURATION` mode. To enable it, simply return to the instance's configuration and set `no shutdown`

```
enable
configure
openflow of-instance 1
no shutdown
```

## Breakout Ports
Requires `CONFIGURATION` mode. The ports will be broken into multiple virtual ports.

```
stack-unit <stack number> port <port number> portmode <mode> speed <speed>
```

Possible `<mode>` and `<speed>` combinations:
  * 1x100G: `single` and `100G`
  * 1x40G: `single` and `40G`
  * 2x50G: `dual` sand `50G`
  * 4x10G: `quad` and `10G`
  * 4x25G: `quad` and `25G`

## Quality of Service (QoS)
There are 2 ways to apply QoS: per-port or per-queue basis. They are mutually exclusive (i.e. if you use per-queue policies, the per-port rate control options disappear.

### Per-Port
Requires `INTERFACE` mode
```
enable
configure
interface gigabitethernet 1/<port number>
```

**Egress** (from the switch) rate shaping:
```
rate shape <mbps> --OR-- rate shape kbps <kbps> (multiples of 64)
```

**Ingress** (into switch) rate policing:
```
rate police <mbps> --OR-- rate police kbps <kbps> (multiples of 64)
```

### Per-Queue
Requires multiple modes: `INTERFACE`, `POLICY-MAP`, `QOS-POLICY`

To summarize the relationships:
  * QoS Policies specify the actual QoS settings (e.g. maximum bandwidth, guaranteed minimum bandwidth)
  * Policy Maps are **abstract** mappings from Queue # to QoS Policies
  * An Interface can then apply a Policy Map to itself

For both QoS Policies and Policy Maps, both Input and Output types exist for both.

**Egress** (from switch) per-queue rate shaping (i.e. maximum bandwidth) and guaranteed minimum bandwidth:
```
enable
configure
qos-policy-output <qos policy name>
rate shape <mbps> --OR-- rate shape kbps <kbps> (multiples of 64)
bandwidth-percentage <percentage>
exit
policy-map-output <policy map name>
service-queue <queue number> qos-policy <qos policy name>
exit
interface tengigabitethernet 1/<port number>
service-policy output <policy map name>
```

**NOTE**: Setting bandwidth percentages for a queue will influence other queues on the same port. Be sure the Policy Map chosen references QoS Policies that add up to less than or equal to 100. If it adds up to over 100, the behaviour is unknown (I assume lower queue #'s take precedence, but haven't tested).

# Troubleshooting and Debugging
This section contains commands for troubleshooting and debugging a Dell switch running Dell Networking OS firmware.

## In-Band Management Issues
  * Management connection to 1GE (IPMI) switch goes through its own data ports (i.e. in-band)
    * Sometimes when management is disconnected briefly, won't come back up
    * Attach to it via serial (should be connected to Core-2 controller), `shutdown` the management interface, and `no shutdown` again

## Issues with 1 GE Adapters
  * MGMT switch is 10 GE, connects to multiple servers (including controller) via 1GE adapters, ofen may see these ports go down
    * This is eiter due to poor support for 1GE negotiation, or poor support for the Finisar FCLF-8521-3 adapters
    * SSH into the management switch, and try:
      * Configure interface to `speed auto`, then `speed 1000` again
      * Configure interface to `no negotiation auto`, see if that works, if not, set it back using `negotiation auto`
      * Try rebooting the switch via `reload`

## Displaying OpenFlow Instance Info
  * To see current configuration, can use: `show openflow of-instance 1` (replace 1 with instance number)
  * To see flow table: `show openflow flows of-instance 1` (replace 1 with instance number)

## Monitoring Interface Stats
  * To continuously monitor, use: `monitor interface tengigabitethernet 1/14` (replace with any other port)
