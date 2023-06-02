---
layout: default
title: Manual VM Migration
section: wikipost
original_date: 2018-02-03
last_modified: 2018-12-23
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified }}*

# {{ page.title }}
This guide shows how to manually migrate VMs across OpenStack agents.
This may be necessary in certain cases (e.g. different hypervisor versions, if VMs use remote storage not shared by the source and destination agents, and etc.).

## Preparation
Gather the following pieces of information before starting migration:
* **Stop the VMs intended for migration**
* VM instances path in both source and destination agents
    * Can probably find this in `/etc/nova/nova.conf` as the `instances_path` parameter
* The backing file for a particular VM's disk
    * Can find this by going into the instance's directory and finding the disk file
    * Issue the command `sudo qemu-img info disk | grep backing`

## General Steps
The general steps are as follows:
1. The backing file must be copied from the source agent to the destination agent in the same path
2. The VM's specific instance directory (named after its UUID) must be copied to the destination agent
3. The VM's entry in nova database should be updated, in particular, the `host` and `node` fields

Copying while preserving permissions and timestamps can be done via `rsync`, e.g.:
* `sudo rsync --rsync-path="sudo rsync" -avzr --progress 3a3abba4-4852-4c04-a42e-94dda4ab80a3 ubuntu@10.12.14.16:/storage/stack/nova/instances/`
