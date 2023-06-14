---
layout: default
title: Removing Hypervisor from OpenStack
section: wikipost
original_date: 2018-03-13
last_modified: 2018-03-13
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
This guide shows how to manually remove a hypervisor (agent compute node) from OpenStack.
Unfortunately, at the time of writing, Nova doesn't allow this in a programmatic fashion, so this has to be done with some mySQL manipulation.

## Step 1: Migrate Existing VMs
Enough said.

If you're having issues with migrating using the existing commands, you may need to do this manually.
See **<a href="manual_vm_migrate.html" target="_blank">this guide</a>** for more details.

## Step 2: Nova Database Manipulation
There're 3 tables in Nova's mySQL database that needs to be updated:
* `compute_nodes`
* `compute_node_stats`
* `services`

First, identify the row ID corresponding to the hypervisor you want to delete:
```sql
select created_at,updated_at,deleted_at,id,deleted,hypervisor_hostname from compute_nodes;
```

You should see something similar to:
```sql
+---------------------+---------------------+---------------------+----+---------+-------------------------+
| created_at          | updated_at          | deleted_at          | id | deleted | hypervisor_hostname     |
+---------------------+---------------------+---------------------+----+---------+-------------------------+
| 2015-06-13 14:47:30 | 2018-03-13 22:55:46 | NULL                |  1 |       0 | agent-1                 |
| 2015-06-13 16:34:59 | 2018-03-13 22:55:49 | NULL                |  2 |       0 | agent-12                |
| 2015-06-13 19:33:02 | 2018-03-13 22:55:48 | NULL                |  3 |       0 | agent-13                |
| 2015-06-13 19:41:46 | 2018-03-13 22:55:46 | NULL                |  4 |       0 | agent-14                |
| 2015-06-13 19:47:30 | 2016-01-15 17:46:18 | NULL                |  5 |       0 | agent-15.openstacklocal |
| 2015-07-04 18:42:00 | 2018-03-13 22:55:45 | NULL                |  6 |       0 | agent-19.openstacklocal |
| 2015-07-04 19:08:15 | 2018-03-13 22:55:47 | NULL                |  7 |       0 | agent-16.openstacklocal |
| 2015-07-04 19:38:07 | 2015-08-15 00:57:06 | 2015-08-15 00:57:10 |  8 |       8 | agent-18.openstacklocal |
| 2015-07-04 19:43:51 | 2018-03-13 22:55:46 | NULL                |  9 |       0 | agent-17.openstacklocal |
| 2015-08-15 00:57:10 | 2015-08-15 00:58:16 | 2015-08-22 08:10:53 | 10 |      10 | agent-20                |
| 2015-08-15 00:58:29 | 2018-02-03 21:38:20 | NULL                | 11 |       0 | agent-20                |
| 2015-08-22 08:10:53 | 2018-03-13 22:55:47 | NULL                | 12 |       0 | agent-18.openstacklocal |
| 2016-08-16 21:05:08 | 2016-11-30 22:17:21 | NULL                | 13 |       0 | vcpe1404.openstacklocal |
+---------------------+---------------------+---------------------+----+---------+-------------------------+
13 rows in set (0.00 sec)
```

In this example, we want to delete **agent-20** which has a row ID of **11**.

### Step 2a: Delete from "compute_nodes" Table
We can now update/delete the entry from `compute_nodes`. In the command below, replace the values of `deleted` and `id` with the row ID previously found.:
```sql
update compute_nodes set deleted_at=now(),deleted=id where id=11;
```

### Step 2b: Delete from "compute_node_stats" Table
Again, in the command below, replace the value of `compute_node_id` with the row ID previously found:
```sql
update compute_node_stats set updated_at=now(),deleted_at=now(),deleted=id where compute_node_id=11 and deleted=0;
```

### Step 2c: Delete from "services" Table
In the command below, replace `host` with the hypervisor name you want to delete:
```sql
update services set deleted_at=now(),deleted=1 where host='agent-20';
```

## Step 3: Verification
Verify by running:
* `nova hypervisor-list`
* `nova host-list`
* `nova service-list`

Can also verify via Horizon dashboard if that's available. The compute resources from the deleted hypervisor should no longer contribute to the total amount of resources reported in the admin panel.

