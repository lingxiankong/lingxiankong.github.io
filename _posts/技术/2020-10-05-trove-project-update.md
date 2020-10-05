---
layout: post
title: OpenStack Trove Project Update
category: 技术
---

> This blog was sent to openstack-discuss mailing list originaly.

As the official Victoria release is approaching and it has been a long time
silence for Trove in the upstream, I think it's good time for me as the Trove
PTL for the last 3 dev cycles to have a project update. The things that will be
described below have not been achieved in one single dev cycle, but are some
significant changes since the 'dark time' of Trove project in the past. Tips
hat to those who have made contributions to Trove project ever before.

## Service tenant configuration

Service tenant configuration was added in Stein release, before that, it's
impossible to deploy Trove in the public cloud (even not for some private cloud
due to security concerns) because the user may have access to the guest
instance which contains sensitive data in the config files, the users can also
perform operations towards either storage or networking resources which may
bring much management overhead and make it easy to break the database
functionality.

With service tenant configuration (which is currently the default setting in
devstack), almost all the cloud resources(except the Swift objects for backup
data) created for a Trove instance are only visible to the Trove service user.
As Trove users, they can only see a Trove instance, but know nothing about the
Nova VM, Cinder volume, Neutron management network, and security groups under
the hood. The only way to operate Trove instances is to interact with Trove API.

## Message queue security concerns

To do database operations, trove controller services communicate with
trove-guestagent service inside the instance via message queue service (i.e.
RabbitMQ in most environments). In the meantime, trove-guestagent periodically
sends status update information to trove-conductor through the same messaging
system.

In the current design, the RabbitMQ username and password need to be configured
in the trove-guestagent config file, which brought significant security concern
for the cloud deployers in the past. If the guest instance is compromised, then
guest credentials are compromised, which means the messaging system is
compromised.

As part of the solution, a security enhancement was introduced in the Ocata
release, using encryption keys to protect the messages between the control
plane and the guest instances. First, the rabbitmq credential should only have
access to trove services. Second, even with the rabbitmq credential and the
message encryption key of the particular instance, the communication from the
guest agent and trove controller services are restricted in the context of that
particular instance, other instances are not affected as the malicious user
doesn't know their message encryption keys.

Additionally, since Ussuri, trove is running in service tenant model in
devstack by default which is also the recommended deployment configuration.
Most of the cloud resources(except the Swift objects for backup data) created
for a trove instance should only be visible to the trove service user, which
also could decrease the attack surface.

## Datastore images

Before Victoria, Trove provided a bunch of diskimage-builder elements for
building different datastore images. As contributors were leaving, most of the
elements just became unmaintained except for MySQL and MariaDB. To solve the
problem, database containerization was introduced in Victoria dev cycle, so
that the database service is running in a docker container inside the guest
instance, trove guest agent is pulling container image for a particular
datastore when initializing guest instance. Trove is not maintaining those
container images.

That means, since Victoria, the cloud provider only needs to maintain one
single datastore image which only contains common code that is datastore
independent. However, for backward compatibility, the cloud provider still
needs to create different datastores but using the same Glance image ID.

Additionally, using database container also makes it much easier for database
operations and management.

To upgrade from the older version to Victoria onwards, the Trove user has to
create backups before upgrading, then create instances from the backup, so
downtime is expected.

## Supported datastores

Trove used to support several datastores such as MySQL, MariaDB, PostgreSQL,
MongoDB, CouchDB, etc. Most of them became unmaintained because of a lack of
maintainers in the community.

Currently, only MySQL and MariaDB drivers are fully supported and tested in the
upstream. PostgreSQL driver was refactored in Victoria dev cycle and is in
experimental status again.

Adding extra datastores should be quite easy by implementing the interfaces
between trove task manager and guest agent. Again, no need to maintain separate
datastore images thanks to the container technology.

## Instance backup

At the same time as we were moving to use container for database services, we
also moved the backup and restore functions out of trove guest agent code
because the backup function is usually using some 3rd party software which we
don't want to pre-install inside the datastore image. As a result, we are using
container as well for database backup and restore.

For more information about the backup container image, see
https://lingxiankong.github.io/2020-04-14-database-backup-docker-image.html.

## Others

There are many other improvements not mentioned above added to Trove since
Train, e.g.

* Access configuration for the instance.
* The swift backend customization for backup.
* Online volume resize support.
* XFS disk format for database data volume.
* API documentation improvement.
* etc.

By the way, Catalyst Cloud has already deployed Trove (in Alpha) in our public
cloud in New Zealand, we are getting feedback from customers. I believe there
are other deployers already have Trove in their production but running an old
version because of previous upstream situation in the past. If you are one of
them and interested in upgrading to the latest, please either reply to this
email or send personal email to me, I would be very happy to provide any help
or guidance. For those who are still in evaluation phase, you are also welcome
to reach out for any questions. I'm always in the position to help in
#openstack-trove IRC channel.
