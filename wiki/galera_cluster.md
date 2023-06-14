---
layout: default
title: Setting Up Galera Cluster
section: wikipost
original_date: 2018-01-02
last_modified: 2019-08-13
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
A Galera Cluster is a multi-master database solution using either MySQL or MariaDB databases.

This document will focus on using MariaDB (in most cases, MariaDB is a straight drop-in replacement for MySQL) due to the fact that it supports Galera out-of-the-box without extra extensions or patches (only true for versions 10.1 and above).

**Note:** Cluster sizes should be odd numbers to avoid split-brain situations. For even-numbered cluster sizes, you'll need an arbitrator (non-database member of cluster) to participate in the voting.

## Pre-Req: Installing MariaDB
MariaDB should be available in package managers, see link below to add repo to fetch specific versions.\\
[[https://downloads.mariadb.org/mariadb/repositories/ | Installing MariaDB from Linux repo]]

**NOTE:** When you get prompted to set a new password, don't enter anything new, just press enter to continue, which should keep the old MySQL root password

If MySQL already exists on the system, MariaDB will remove it and replace it with itself. All shell commands to "mysql" will effectively be commands to MariaDB, so no need to change any existing scripts. All software client libraries for MySQL should also still work with MariaDB.

## Enabling Galera
Galera 3 is included in all installations of MariaDB since version 10.1. Enabling it simply involves some extra configuration options.
1. `cd` to the `/etc/mysql/conf.d` directory
2. Create a new file called `galera.cnf`
3. Open the file and use the following template to fill it, adjusting the parameters as needed:
    *
    ```
    [mysqld]
    binlog_format=ROW
    default-storage-engine=innodb
    innodb_autoinc_lock_mode=2
    bind-address=0.0.0.0
    
    # Galera Provider Configuration
    wsrep_on=ON
    wsrep_provider="/usr/lib/galera/libgalera_smm.so"
    
    # Galera Cluster Configuration
    wsrep_cluster_name="galera cluster name"
    wsrep_cluster_address="gcomm://192.168.0.10,192.168.0.20,192.168.0.30?pc.wait_prim=no"
    
    # Galera Synchronization Configuration
    wsrep_sst_method=rsync
    wsrep_sst_receive_address="192.168.0.10:4444"
    
    # Galera Node Configuration
    wsrep_node_address="192.168.0.10:4567"
    wsrep_node_incoming_address="192.168.0.10:3306"
    wsrep_node_name="Node-1"
    
    # If behind a NAT, uncomment the line below and specify the private address
    #wsrep_provider_options="ist.recv_bind=172.17.0.2"
    
    wsrep_debug=1
    ```
    * **NOTE 1:** The following parameters above are only needed if you're changing default port numbers or dealing with NATs where you have to advertise a different public IP to other nodes
        * `wsrep_sst_receive_address` has default port 4444
        * `wsrep_node_address` has default port 4567
        * `wsrep_node_incoming_address` has default port 3306
    * **NOTE 2:** The `wsrep_cluster_name` parameter **must be the same in all nodes**, otherwise a joining node with a different name will fail to start
4. Start the node. This will differ depending on if this is the first node of the cluster (i.e. the bootstrap node) or a node joining an existing cluster
    * If this is the bootstrap node, run: `service mysql start %%--%%wsrep-new-cluster`
    * If this is a joining node, start the node as usual: `service mysql start`
    * **NOTE 1:** To help debug errors, monitor the mysql logs (may be defaulted to syslog) in both the joiner and donor nodes
    * **NOTE 2:** If the node successfully started, you can check the cluster status: `mysql -uroot -p -e "SHOW STATUS LIKE 'wsrep%';"`

## Troubleshooting
You may encounter the following message when using Linux `service` to start/stop/restart or check the status of the database:\\
`ERROR 1045 (28000): Access denied for user 'debian-sys-maint'@'localhost' (using password: YES)`

This message indicates that the maintenance user's password stored in `/etc/mysql/debian.cnf` is out of sync with the password in the database itself. If you just added your database as a new node of a replicated cluster, then **this is expected behaviour**, as the password from the original bootstrap node has now been replicated and replaced your original maintenance password. To fix this, find the bootstrap node (or any node where you don't see the error message when querying the database status), and copy the password from `/etc/mysql/debian.cnf` from that node to the current node.

If your database is not part of a cluster, then you need to update the password in the database by doing the following:
  - Get the default password for user debian-sys-maint
    * `cat /etc/mysql/debian.cnf  | grep -i password`
  - Access the database shell
    * `mysql -uroot -p`
  - From the database shell, change the password for user debian-sys-maint
    * `SET PASSWORD FOR 'debian-sys-maint'@'localhost' = PASSWORD('**PASSWORD-FROM-STEP-1**');`
    * quit
  - Try restarting the database again, you shouldn't see the error anymore
    * `sudo service mysql restart`

## Useful Links
<a href="https://mariadb.com/kb/en/library/getting-started-with-mariadb-galera-cluster/" target="_blank">Getting Started with MariaDB Galera Cluster</a>

<a href="http://galeracluster.com/documentation-webpages/quorumreset.html" target="_blank">Resetting quorum within the cluster</a>

<a href="http://galeracluster.com/documentation-webpages/galeraparameters.html" target="_blank">Galera parameter</a>

<a href="http://galeracluster.com/documentation-webpages/firewallsettings.html" target="_blank">Galera firewall settings (ports to open)</a>

<a href="http://galeracluster.com/documentation-webpages/mysqlwsrepoptions.html" target="_blank">Galera WSREP parameters</a>

<a href="https://severalnines.com/blog/how-bootstrap-mysqlmariadb-galera-cluster" target="_blank">Different scenarios of how to start/re-start a cluster</a>

<a href="http://galeracluster.com/documentation-webpages/quorumreset.html" target="_blank">Resetting quorum of a corrupted cluster</a>

