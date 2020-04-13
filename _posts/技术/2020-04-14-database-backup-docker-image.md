---
layout: post
title: Database Backup/Restore Docker Image
category: 技术
---

Now you have your database (mysql, mariadb, etc.) up and running, either directly on a host or inside a docker container. The next step you may consider is how to perform management tasks, e.g. backup and restore. It is important to back up your databases so that you can recover your data and be up and running again in case problems occur, such as system crashes, hardware failures, or users deleting data by mistake. Backups are also essential as a safeguard before upgrading a MySQL installation, and they can be used to transfer a MySQL installation to another system or to set up replication slave servers.

You may have several requirements like below:

* Install as less extra tooling as possible.
* Support different types of backups, e.g. full backup and incremental backup.
* Ability to choose the backup destination, e.g. local or remote.
* Support backup options like compression, encryption, etc.
* Last but not least, it's preferable to use open source solution rather than enterprise service.

The docker image I am gonna introduce here could tick all the above boxes, the image is `openstacktrove/db-backup`.

This image was developed initially for [OpenStack Trove project](https://docs.openstack.org/trove/latest/), but it can be used anywhere with access to the database.

## Features

* Runs in a separated container
* Support both full backup and incremental backup.
* Pluggable non-blocking backup utilities such as XtraBackup, Mariabackup, etc.
* Pluggable destination storage such as local, OpenStack Object Storage service, etc.
* Automatically pick up all the parent backups when restoring from an incremental backup.

## Swift examples

### Prerequisites

* You have the user credentials to connect to the database server and have permission to perform operations on the server
* You have the openstack user credentials to access the object storage service (Swift).

### Create a full backup

```
docker run --rm --name db-backup --network=host -v /var/lib/mysql:/var/lib/mysql openstacktrove/db-backup:mysql-5.7 /usr/bin/python3 main.py \
--backup --backup-id=<UUID> --storage-driver=swift --driver=innobackupex \
--db-user=<Database Username> --db-password=<Database Password> --db-host=127.0.0.1 \
--os-token=<OpenStack User Token> --os-auth-url=http://openstack.service/identity/v3 --os-tenant=<OpenStack Tenant Name>
```

* `--backup-id`: The customized indentifier for the backup, e.g. a UUID.
* `--db-xxx`: The database user credentials.
* `--os-xxx`: The openstack user credentials.

After the container completing the task successfully, you can check the backup objects in Swift via running the following command:
```
openstack object list database_backups
```

> NOTE: `database_backups` is the default swift container name used by the backup container.

### Create an incremental backup

```
docker run --rm --name db-backup --network=host -v /var/lib/mysql:/var/lib/mysql openstacktrove/db-backup:mysql-5.7 /usr/bin/python3 main.py \
--backup --backup-id=<UUID> --incremental --storage-driver=swift --driver=innobackupex \
--db-user=<Database Username> --db-password=<Database Password> --db-host=127.0.0.1 \
--os-token=<OpenStack User Token> --os-auth-url=http://openstack.service/identity/v3 --os-tenant=<OpenStack Tenant Name> \
--parent-location=http://objectstorage.openstack.service:8080/v1/AUTH_25739c69c0e34841abb340de8b440a24/database_backups/1000.xbstream.gz \
--parent-checksum=f07fe1d42b5beda170d8adb192a530a1
```

* `--parent-location`: The parent backup object URL in OpenStack Swift.
* `--parent-checksum`: Checksum of the parent backup object, for validation purpose.

### Encrypted backups

If you want to create encrypted backups, just simply add `--backup-encryption-key [You Password Here]`, remember to add the same parameter for restore.

### Restore backup

```
docker run --rm --name db-backup --network=host -v /var/lib/mysql:/var/lib/mysql openstacktrove/db-backup:mysql-5.7 /usr/bin/python3 main.py \
--nobackup --storage-driver=swift --driver=innobackupex \
--os-token=<OpenStack User Token> --os-auth-url=http://openstack.service/identity/v3 --os-tenant=<OpenStack Tenant Name> \
--restore-from=http://objectstorage.openstack.service:8080/v1/AUTH_25739c69c0e34841abb340de8b440a24/database_backups/1000.xbstream.gz \
--restore-checksum=f07fe1d42b5beda170d8adb192a530a1
```

* `--nobackup`: Means we are going to perform restore operation rather than backup.
* `--restore-from`: The object URL in OpenStack Swift of the backup we are going to restore from.
* `--restore-checksum`: Checksum of the backup object, for validation purpose.

## Contact

* Name: Lingxian Kong
* Email: anlin.kong@gmail.com

Any feedback or contribution is welcome.
