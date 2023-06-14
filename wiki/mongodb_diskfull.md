---
layout: default
title: Removing Old Data in MongoDB when Disk is Full
section: wikipost
original_date: 2017-07-06
last_modified: 2017-07-06
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

##### *Last updated: {{ page.last_modified | default: page.original_date }}*

# {{ page.title }}
Clearing old data and freeing space back to the OS requires extra space (unless WiredTiger storage engine is in use). This presents a catch-22... to clear space, you need space!

Work-around: Attach a volume and use that as the repair path for MongoDB. We'll do the following:
  - Create a temporary volume of 40 GB (replace 40 with whatever you need)
  - Attaches it to your VM of choice
  - From within the VM:
    - Format volume (double-check to see if it's actually /dev/vdb) to ext4
    - Mount the volume as a **sub-directory** of the data directory (this is a must)
    - Run the clean/repair command
    - Unmount the volume
  - Delete the volume after

### Creating the volume
From a client machine
```bash
nova volume-create --display-name mongotmp 40
nova volume-attach <VM UUID> <Volume UUID>
```

### Deleting the old data
From within the VM
```bash
sudo mkfs.ext4 /dev/vdb # Double-check vdb is what it was mounted as, change as needed
sudo mkdir /var/lib/mongod/tmp # Assumes /var/lib/mongod is the path of the database, change as needed
sudo mount /dev/vdb /var/lib/mongod/tmp

sudo service mongod stop # Do a ps aux and make sure its dead
mongod --dbpath /var/lib/mongod/ --repair --repairpath /var/lib/mongod/tmp
sudo service mongod restart # Do a ps aux and make sure its alive again

df -h # Verify you have free space in your main drive
sudo umount /var/lib/mongod/tmp
```

### Deleting the volume
From a client machine
```bash
nova volume-detach <VM UUID> <Volume UUID>
nova volume-delete <Volume name or UUID>
```

