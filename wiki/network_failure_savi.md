---
layout: default
title: Possible Causes for Network Failures in SAVI
section: wikipost
last_modified: 2020-12-23
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified }}*

# {{ page.title }}

SAVI's network is built upon a mix of Quantum/Neutron's out-of-the-box OVS configuration + custom network logic via SDN.

The following is a checklist of possible causes when there are connectivity issues in the network.

* Routing table
	* Entry for destination network exists?
	* Any conflicts between entries?
	* Target output interface is correct?
* Interface issues
	* Does it exist? (sometimes they're OVS internal interfaces)
	* Is it "up"?
	* Link status? (Only applies if it's a physical interface on the server)
		* Can check using `ethtool`
	* Does it have the correct IP & netmask set?
	* Is it the correct physical interface?
		* Sometimes upon server bootup, the eth* names gets shuffled
		* Important to have the MAC address of each interface documented, check against this document to verify
* OVS issues
	* If the OVS is br-int, is it connected to a controller?
	* If it's OpenFlow-controlled, `dump-flows` and grep for src/dst MACs or IPs
	* If OVS is used as a simple switch, `dump-flows` and make sure `action=NORMAL`
	* If a matchable flow is identified, check to see if counters go up as expected
* If `tcpdump` shows packets are existing one server but not being received by the other, check physical OpenFlow switch
	* In controller server, do `netstat -tpn | grep 6633`
		* See if it has a connection to the physical switch's IP with status `ESTABLISHED`
		* Pronto-based firmware known to sometimes disconnect OVS from controller and not re-establish, requires **reboot of switch**
	* Check if physical interface of switch is enabled
	* If dealing with 10 GE ports, ensure cable bandwidth and port settings match
		* Sometimes we use 1 GE cable with 10 GE cable, in this case port may need to be explicitly set to 1 GE rate
		* Sometimes port set to 1 GE rate while using 10 GE cable
* OpenFlow issues (for OF-enabled switches)
	* If OpenFlow connection is fine (i.e. shows `ESTABLISHED` in `netstat`), check Ryu logs for errors
		* Try restarting Ryu
	* Check Janus logs for errors
		* Try restarting Janus
* VMs can't get IPs or ping another VM
	* `tcpdump` or do `arp -a` to see if ARP is the issue
		* Try restarting ARP handler in controller
	* Sometimes `dnsmasq` (DHCP server) in controller goes into bad state, **difficult to debug**
		* Try restarting n-dhcp in controller
* VMs can't connect to the internet **(VM => Internet/Anyone else)**
	* Try pinging both 8.8.8.8 and/or 8.8.4.4 to check internet connectivity
	* If no internet connectivity, check Neutron Router namespace in controller
		* Check (1), (2), and (3) within the router
	* If internet connectivity fine, check VMs `/etc/resolv.conf`
		* If it's not set, quick fix is to manually edit it
		* Root cause may be `dnsmasq` issue or improperly set Neutron Subnet settings
* Can't connect to VMs **(Internet/Anyone else => VM)**
	* Check security group of VM via `nova show`
	* Check security group rules of the group(s) assigned to VM
	* Check which SAVI edge the VM is on
		* Carleton and Postech known to block ICMPs and certain other ports
* Keystone Connection Issue **(York Region)**
	* There is a ssh tunnel service that needs to be up and running: sudo /etc/init.d/autossh status
	* If it's not up, restart that service. The python and shell scripts are in /home/savi/iam_check 

## Debugging OVS Flow Tables
Can simulate a packet using ovs-appctl (will not actually create a packet, so table stats won't change). This is useful for debugging large tables and seeing which flow(s) will match.
```bash
sudo ovs-appctl ofproto/trace <bridge name> <flow spec> -generate
```

Example:
```bash
savi@agent-24:~$ sudo ovs-appctl ofproto/trace br-int arp,in_port=13,dl_src=fa:16:3e:77:bf:cd,dl_dst=ff:ff:ff:ff:ff:ff -generate
Bridge: br-int
Flow: arp,in_port=13,vlan_tci=0x0000,dl_src=fa:16:3e:77:bf:cd,dl_dst=ff:ff:ff:ff:ff:ff,arp_spa=0.0.0.0,arp_tpa=0.0.0.0,arp_op=0,arp_sha=00:00:00:00:00:00,arp_tha=00:00:00:00:00:00

Rule: table=0 cookie=0 priority=62768,arp,in_port=13,dl_src=fa:16:3e:77:bf:cd,dl_dst=ff:ff:ff:ff:ff:ff
OpenFlow actions=mod_dl_dst:fa:16:3e:1e:ea:7d,output:1

Final flow: arp,in_port=13,vlan_tci=0x0000,dl_src=fa:16:3e:77:bf:cd,dl_dst=fa:16:3e:1e:ea:7d,arp_spa=0.0.0.0,arp_tpa=0.0.0.0,arp_op=0,arp_sha=00:00:00:00:00:00,arp_tha=00:00:00:00:00:00
Megaflow: recirc_id=0,arp,in_port=13,dl_src=fa:16:3e:77:bf:cd,dl_dst=ff:ff:ff:ff:ff:ff,arp_spa=0.0.0.0,arp_tpa=0.0.0.0,arp_op=0
Datapath actions: set(eth(src=fa:16:3e:77:bf:cd,dst=fa:16:3e:1e:ea:7d)),1
```
