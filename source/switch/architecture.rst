SWITCHdrive Architecture
========================


The following picture shows the current architecture diagram.

.. image:: images/architecture/Architekturdiagramm0.7.png


The current architecture is not HA (High Availability) enabled. As all the VMs
are running on physical hosts, if one of them breaks, then potentially multiple
VMs are affected at the same time. For now, I choose the path of being able to
restart the VMs on another physical host in case of failure.

Common
------

All virtual machines are running Ubuntu 13.10 in a standard server installation.
They have IPtables installed and only expose TCP port 22 plus the ports necessary
for the service to the internal 10.0.20.0/24 network.

Loadbalancer
------------

Incoming requests are handled by HAProxy_. HAProxy
does load balancing and SSL termination. Doing SSL termination on the
load balancer allows it to inject cookies into the HTTP traffic that enables it
to provide "sticky sessions" (i.e. requests from a single client always go to
the same web server).

The HAProxy log files are sent to the syslog server.

HAProxy runs completely in memory and doesn't touch the disks while running. It
uses a single-threaded evented execution model and therefore is quite happy on a
2 VCPU machine. To increase performance, it helps to pin the virtual CPUs to
physical CPUs on the host running the VM.

Open Ports:

  * 22
  * 80
  * 443

Web/Application servers
-----------------------

The web/application servers are handled by nginx_ and `PHP5 FPM`_.

nginx handles all incoming HTTP requests from the load balancer, serves
static files (css,js as well as the actual files stored in ownCloud) and passes
all requests to ownCloud to the PHP5-FPM task.

nginx is configured with 32 worker processes
(http://wiki.nginx.org/CoreModule#worker_processes) in order to maximize the
amount of files that can be sent when waiting for disk io.

PHP-FPM spawns a number of PHP worker processes on startup and manages them
automatically (spawning more if the load rises and killing them again if load
decreases). The goal is to have a PHP process ready whenever a request comes in.
(The process manager will always keep 8 PHP processes running and spawn up to 30
processes when under load)

Each web server mounts the ownCloud data directory as ``/mnt/data`` from the NFS
server.

The ownCloud logfiles are sent to the syslog server

The web servers are 8 VCPU, 16GB RAM virtual machines. The amount of RAM is so
high because ownCloud allows large files to be uploaded through the web UI. The
maximal file size is 4GB, but during the upload, the file is in memory. Lower
amounts of RAM could lead to situations where the memory on the server is being
exhausted.

Open Ports:

  * 22
  * 80

Database Server
---------------

Instead of MySQL we have opted to use PostgreSQL (currently version 9.3) to
store the database.

The database is backed up every 6 hours via a crontab entry. Future improvments
will include DB snapshot and WAL archiving.

ownCloud is read heavy (factor 1000:1) so a possible performance option will be
to have Postgres read slaves and a Postgres aware load balancer
`PgPool II`_ to separate write and read requests.

The VM is configured with 32 GB of RAM and 8 VCPUs. The database currently is
tuned for those settings, so resizing the VM needs to take that into account
(both postgres.conf and the virtual memory settings of the server need to be
adapted)

Open Ports:

  * 22
  * 5432

Database Slave Server
---------------------

We are using Postgres streaming replication to keep a hot standby database
server. The slave receives all changes from the master server and is able to

  * take over the role of the master in case of emergency (procedure
    needs to be described)
  * act as a read slave to distribute read load

Open Ports:

  * 22
  * 5432

NFS Server
----------

OwnCloud needs access to a file system where it stores the users files. Because
we have multiple web servers and each server needs access to the same file
system, we have a 20 TB volume that is mounted in a VM and exposed with an NFS
server.

The 20TB volume is a RBD_ volume backed
by Ceph_. It is thin provisioned and can be resized to
accomodate for future growth.

An additional 20TB volume is used for generational file system backups with
RSnapshot_. This is mainly a safeguard for the
case when ownCloud should lose data.

There hasn't been much tuning of the NFS server yet (save for mounting the
volumes asynchronously and enabling RBD caching for the VM). The clients use
1MB large read and write buffers when mounting the NFS server. Benchmarks have
shown that the NFS server can write around 120MB/s, which should be enough for
now.

On initial syncs or rapid reads of ownCloud data, we notice huge spikes of CPU
activity. Should that turn out to be a problem, we can potentially throttle the
IOPS that the VM is allowed to perform - that could make an improvement in CPU
load (lower overall IOPS but sustained performance even under load)

Open Ports:

  * 111 (tcp/udp)
  * 2049 (tcp/udp)

LDAP Server
-----------

The OpenLDAP server provides the authentication service for the ownCloud
installation (and also for other SWITCH cloud based services). It is a single
LDAP server with a directory structure adapted to the needs of the different
cloud projects.

The database of the LDAP server is backed up daily in .ldif format (into
``/var/backup/slapd``

Open Ports:

  * 22
  * 636

Syslog Server
-------------

The rsyslog server collects the logfiles from ownCloud (the application) and the
access logs from the haproxy server.

Those files are stored on a separate 100GB volume, mounted at ``/var/log``

Open Ports:

  * 514

CloudID Server
--------------

The cloud id server is used to bridge between AAI and the LDAP server. External
users can login with AAI and are able to create a new account (that will be
commissioned on the LDAP server) or to reset their password.

The server runs a Ruby on Rails application developed by SWITCH's
Interaction Enabling team. It uses Apache and mod_shib, and xxx as its
database.

Open Ports:

  * 22
  * 80
  * 443

Monitoring Server
-----------------

The monitoring server runs Zabbix_ and collects statistics
from the different virtual machines and provides statistics and graphs.

It runs Apache with PHP and Postgres. While the initial configuration of the
Zabbix server is done automatically, the actual configuration of monitored
servers is done manually.

Open Ports:

  * 22
  * 443
  * 10050
  * 10051

Thoughts on High Availability
-----------------------------

The current setup is not HA (high-availability) one. While it certainly would be
possible to build a complete HA setup, we have decided against this for a number
of reasons:

  * We don't expect the virtual machines to fail. If hardware fails, it will be
    the physical hypervisors. In the current (smallish) deployment, there are not
    enough machines to make the failure of just one not taking down multiple of
    the ownCloud VMs.
  * IP failover in the OpenStack environment is a bit complicated (we can use
    the ``nova`` command line API to switch a floating IP to another VM, but this
    is not well integrated with the common HA solutions (``hearbeat/corosync`` or
    ``keepalived``

In case of failure of a physical host or a VM, we are prepared to experience some
downtime. In case of a failed host, the dead VMs can be restarted on another physical
host or rebuilt using the ansible scripts within a few minutes.



.. links

.. _HAProxy: http://haproxy.1wt.eu/
.. _nginx: http://nginx.org/
.. _`PHP5 FPM`: http://php-fpm.org/
.. _PostgreSQL: http://www.postgresql.org/
.. _`PGPool II`: http://www.pgpool.net/mediawiki/index.php/Main_Page
.. _RBD: http://ceph.com/docs/master/rbd/rbd/
.. _Ceph: http:/ceph.com
.. _RSnapshot: http://www.rsnapshot.org/
.. _Zabbix: http://zabbix.org
