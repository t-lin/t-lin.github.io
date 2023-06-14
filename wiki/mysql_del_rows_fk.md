---
layout: default
title: Deleting Rows in MySQL w/ Foreign Key Constraints
section: wikipost
original_date: 2018-05-31
last_modified: 2018-05-31
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
OpenStack often uses "soft deletion" in its MySQL database, which simply sets a "deleted" flag to a non-zero value. This makes the database grow to very large sizes.

When trying to prune the database and delete rows in a given table, may run into issues where other tables have rows that references the current table (foreign key constraint).

To get around it, can simply drop all foreign key constraints, but this is dangerous and can lead to inconsistent database state or leave unused rows in other tables.
Instead, re-create the constraints with `ON DELETE CASCADE` option.

General steps:
1. Find other tables, see how their constraint was set up
    * ```sql
SHOW CREATE TABLE <table name>
```
    * Note the `CONSTRAINT` line, and specifically the name of the constraint
2. Drop the constraint
    * ```sql
ALTER TABLE <table name> DROP FOREIGN KEY <constraint name>
```
3. Re-add the constraint but with `ON DELETE CASCADE` option
    * ```sql
ALTER TABLE <table name> ADD <entire constraint line> ON DELETE CASCADE
```
4. Then try deleting rows from the original table again. Reiterate above steps if any other constraints show up.

Example of steps 1-3 from above:
```sql
mysql> SHOW CREATE TABLE instance_faults;
+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table           | Create Table                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| instance_faults | CREATE TABLE `instance_faults` (
  `created_at` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT NULL,
  `deleted_at` datetime DEFAULT NULL,
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `instance_uuid` varchar(36) DEFAULT NULL,
  `code` int(11) NOT NULL,
  `message` varchar(255) DEFAULT NULL,
  `details` mediumtext,
  `host` varchar(255) DEFAULT NULL,
  `deleted` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `instance_faults_host_idx` (`host`),
  KEY `instance_faults_instance_uuid_deleted_created_at_idx` (`instance_uuid`,`deleted`,`created_at`),
  CONSTRAINT `fk_instance_faults_instance_uuid` FOREIGN KEY (`instance_uuid`) REFERENCES `instances` (`uuid`)
) ENGINE=InnoDB AUTO_INCREMENT=1288 DEFAULT CHARSET=utf8 |
+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> ALTER TABLE instance_faults DROP FOREIGN KEY fk_instance_faults_instance_uuid;
Query OK, 1253 rows affected (0.09 sec)
Records: 1253  Duplicates: 0  Warnings: 0

mysql> ALTER TABLE instance_faults add CONSTRAINT `fk_instance_faults_instance_uuid` FOREIGN KEY (`instance_uuid`) REFERENCES `instances` (`uuid`) ON DELETE CASCADE;
Query OK, 1253 rows affected (0.13 sec)
Records: 1253  Duplicates: 0  Warnings: 0
```

