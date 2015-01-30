CESNET Architecture
===================

The following picture shows the current architecture of the
ownCloud service at CESNET.

.. image:: images/architecture/CesnetOcArchitecture1.0.png

Our ownCloud architecture is built on top of HA (High Availability) cluster
managed by Pacemaker_. The cluster consists of 5 equivalent nodes.
Pacemaker ensures that ownCloud and other relevant services are running
properly even if one or more front-end dies. All nodes are physical machines,
currently there is no virtualization. Key feature of our architecture
is that all nodes are equal as each of them can take any role (be it an app server or a DB server).

Our system consists of the following components and services:

* Application server
* Database server
* PgPool proxy
* Pacemaker HA services
* Icinga & Munin monitoring servers
* eduID and eduGAIN authentication services
* GPFS storage
* TSM server for backups

Specs
------

Hardware specifications of each front-end are as follows:

  * CPU: 2 x Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz (24 CPUs)
  * RAM: 96 GB (~ 40 GB ownCloud-dedicated)
  * Network: 2 x 10 GE (bonded)

GPFS is being used as an underlying filesystem. OwnCloud utilizes a dedicated
40 TB volume which is mounted on all front-ends. The IBM DCS3700 disk array
is being used for data storage.

All front-end nodes are running Red Hat Enterprise Linux 6.

Application Server
------------------

OwnCloud application is handled by Apache_ 2.2 and PHP_ 5.4
(which has been installed from non-standard repository -- see this guide_).

Newer PHP version was used instead of PHP 5.3 version (which is default on RHEL 6) to allow
uploads of files larger than 2 GB. There were problems with this in 5.3.X series.
We are able to do up to 16 GB uploads on PHP5.4 with no problems at all.

Apache is running as a standalone server without any HTTP(S) proxies in the way. It is
listening on ports 80 and 443. HTTPS is being forced from 80 to 443 by
redirection.

Complete Apache + PHP configuration and website root is located on a shared GPFS volume,
which is mounted on all cluster nodes.

Apache is configured to use the default **prefork** MPM module for handling requests.
However, its configuration has been tweaked to match nodes specs and current load::
        <IfModule prefork.c>
        StartServers         500
        MinSpareServers       10
        MaxSpareServers       30
        ServerLimit         1800
        MaxClients          1600
        MaxRequestsPerChild 8000
        </IfModule>

Since we have quite a lot of RAM available on the servers, we increased the maximum number of workers
to 1600, so it uses up to 40 GB of RAM with some reserve (by our measurements, one Apache
worker process takes 22 MB in average).
The **StartServers** directive is set to a quite high value, too, in order
to handle peak loads after server restarts. In such a situation,
user's clients start reconnecting massively, overloading the server significantly for
a short period of time.

Apache is set up with `Zend OPcache`_ module for better PHP performance and `mod_xsendfile`_, which is being used for faster file downloads (also provides users with pause/resume download capabilities).

Open Ports:

  * 80
  * 443

Database Server
---------------

PostgreSQL_ 8.4 is being used as a database server of choice. This is the highest version
available in standard RHEL 6 repositories. The database server runs in a single instance, preferably on a
different node than the Apache instance and without any replication yet. We are planning
an upgrade to 9.4 and a hot-standby streaming replication setup with load balancing. We expect
this to help us in the future with a read heavy nature of ownCloud queries, but as of now the current setup copes with the load just fine.

OwnCloud PHP application doesn't access the PostgreSQL instance directly. It sends its queries
through `PgPool II`_, which acts as a connection cache (running in *Connection Pooling* mode).
This gives us some performance boost as it reduces database instance load by means of reusing existing connections (thus saving cost of creating new ones). PgPool is run together with the Apache instance on the same node.

PostgreSQL engine parameters are mostly set according to the pgtune_ utility recommendations.
It has been allocated 40 GB of RAM, the same as the Apache instance, and a maximum number of connections is set to **51** (mostly based on recommendations_ in PostgreSQL documentation).

The database server keeps its data and configuration on a shared GPFS volume just like the Apache server.

Open Ports:

  * 5432
  * 5433 (PgPool II)

High Availability
-----------------

Pacemaker_ 1.1 on top of RedHat's CMAN 3.0 is used as High Availability
resource manager for all services and applications. Main advantages of this setup are
  * defined order in which individual resources are started,
  * default locations where services are run,
  * automatic distribution of services based on their mutual linkage,
  * periodical checks if all services are running,
  * and automatic failover.
    
We use basic Resource Agents (RAs) available from system repository for controlling Apache,
IP aliases and PostgreSQL services. Due to the lack of repository RA for PgPool II, we
developed our own RA script.
From the ownCloud point of view, we use six resources, all of them run in active-passive
mode. First of all, PostgreSQL DB engine is started on a node and an IP
alias for database is configured. Start of the PgPool II RA is the next step, which takes place on a
different node than the DB. Both IPv4 and IPv6 aliases for ownCloud service are started on the node as soon as the PgPool II is running, and finally the Apache RA is started. Pacemaker
guarantees shutdown of all services in a contrary order if necessary.

.. _samlfix:

User Authentication
-------------------

Authentication of users is based on SAML. It relies on the SimpleSAMLphp_ backend application for 
authentication and providing user's metadata. SimpleSAMLphp backend is configured with eduID_ IdP (Identity Providers) metadata and acts like an SP (Service Provider) in the federations. 
When users try to log in, they are presented with a WAYF_ page, where they can pick their home 
organizations. They are then redirected to their organization's IdP login page where they log in.
After a successful log in, we get all necessary information about a user (EPPN, e-mail) from user's home organization IdP.

When we were looking for a solution of user authentication, there were two available
user backends for ownCloud, which allowed federated user accounts to log in -- `user_saml`_ and `user_shibboleth`_. Both of them were quite outdated and not working well in ownCloud 6, however.
We have picked the *user_saml* app and fixed an issues it had with OC 6 in this `pull request`_.
Without those fixes, user creation and logout was broken. That way only already existing ownCloud
users could log in using SAML authentication and the 'Logout' option from the menu did nothing.

Data Storage and Backup
-----------------------

All the data is stored in a dedicated GPFS filesystem mounted on all nodes, so all
nodes in the cluster can access the same data. For this filesystem, we reserved 40TB of disk
space. The filesystem is built on top of 4 RAID6 groups from IBM DCS3700 disk array, which is connected
through Fibre Channel infrastructure to all frontend nodes. We use this filesystem to store PostgreSQL database datafiles, ownCloud
user data, web interface files for the webserver, as well as logging of all installation components.

Data backups are realized by a GPFS utility mmbackup. This utility scans the whole filesystem
(using GPFS inode scan interface) and passes a changed, new or deleted files to TSM (Tivoli Storage 
Manager) server. TSM then runs selective (full) backup (or expiration when file deleted) on those files. We retain a history of 2 versions of the backed files in TSM for 30 days. TSM is being used with IBM TS3500 tape library as a persistent storage device for holding backups. OwnCloud backups are run periodically once a day.

Before each backup run, PostgreSQL database is being dumped using pg_dump utility.
Pg_dump generates the archive and mmbackup then finds this file on the GPFS filesystem
and sends it to TSM with the rest of ownCloud files to be backed up.

Monitoring
----------

All ownCloud specific services are constantly monitored by Icinga_ (a fork of Nagios).
We had to write own custom plugins to check some ownCloud specific stuff.
Following items are being periodically checked by Icinga:

  * SSL certificate validity
  * WebDAV client functioning properly (data can be uploaded and downloaded)
  * Free space on OC GPFS volume
  * Apache responding on HTTPS
  * PING (machine with owncloud-ip responding)
  * PostgreSQL (Postgres is responding on postgres-ip and OC can connect to the database)

In addition to this, we use custom Munin_  plugins to collect usage statistics
and create graphs. Currently we are graphing the following ownCloud statistics:

  * Number of user accounts
  * Number of files
  * Amount of user data stored
  * Apache response times
  * Bytes transferred by Apache
  * Filesystem space used

We are also collecting all relevant logs to a central server, where it could be
further analyzed and queried with LogStash and ElasticSearch.

.. links
.. _Pacemaker: http://clusterlabs.org/quickstart-redhat.html
.. _Apache: https://httpd.apache.org/
.. _PHP: http://www.php.net/
.. _guide: http://developerblog.redhat.com/2013/08/01/php-5-4-on-rhel-6-using-rhscl/
.. _`Zend OPcache`: http://pecl.php.net/package/ZendOpcache
.. _`mod_xsendfile`: https://tn123.org/mod_xsendfile/
.. _PostgreSQL: http://www.postgresql.org/
.. _`PgPool II`: http://www.pgpool.net/mediawiki/index.php/Main_Page
.. _pgtune: http://pgtune.leopard.in.ua/
.. _recommendations: http://wiki.postgresql.org/wiki/Number_Of_Database_Connections#How_to_Find_the_Optimal_Database_Connection_Pool_Size
.. _SimpleSAMLphp: https://simplesamlphp.org/
.. _eduId: http://eduid.cz/
.. _eduGAIN: http://www.geant.net/service/eduGAIN/Pages/home.aspx
.. _`user_saml`: https://github.com/owncloud/apps/tree/master/user_saml
.. _`user_shibboleth`: https://github.com/AndreasErgenzinger/user_shibboleth
.. _WAYF: https://www.eduid.cz/en/tech/wayf
.. _Icinga: https://www.icinga.org/
.. _Munin: http://munin-monitoring.org/
.. _`pull request`: https://github.com/owncloud/apps/pull/1681
