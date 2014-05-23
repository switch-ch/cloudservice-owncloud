Installation
============

Prerequisites
-------------
 
Software
^^^^^^^^

You will need to have Git_ and Ansible_ 1.4/1.5 installed on your
local machine. `Ansible installation`_ takes you through the steps to get
Ansible up and running on your local machine.

SSH configuration
^^^^^^^^^^^^^^^^^

If you use an OpenStack-based cloud environment, the various virtual machines
won't have any public IP addresses (except for the load balancer). In order to
get SSH access to the VMs, you will need to do a bit of configuration on a
jumposthost  SWITCHcloud to host the virtual machines, you will need to add
the following to your ``~/.ssh/config`` to get direct SSH access to the
10.0.20.0/24 net (or whatever it is in your configuration) the VMs run on::

  Host *.example.ch
    ForwardAgent yes


  Host 10.0.20.*
    LogLevel error
    ForwardAgent no
    ForwardX11 no
    CheckHostIP no
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
    ProxyCommand ssh jump.cloud.example.com nc %h %p
    controlmaster auto
    ControlPath /tmp/ssh_mux_%h_%p_%r

Also, you will need to have a user account on ``jump.cloud.example.com`` (or
whatever your host is called).

Clone Ansible project
^^^^^^^^^^^^^^^^^^^^^

Clone the ansible project on your local machine::

  $ cd work
  $ git clone github.com:/to/be/defined.git

Anatomy of the Ansible project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The project is set up with several folders and files:

  * backups - will hold the downloaded database backup files (empty at start)
  * group_vars - variables for the different server groups (mostly open tcp/udp ports)
  * library - additional ansible tasks
  * roles - the meat of the configuration. Contains the setup instructions for the different roles
  * ssl - storage for SSL certificates and keys (not checked in)

Files:

  * staging - IP addresses and config variables for **staging** environment
  * production - IP addresses and config variables for **production** environment
  * db,dev,lb,ldap,nfs,syslog,web}servers.yml - specification of the various server classes
  * various commands for [[switchdrive:commonoperations|operating]] the cluster

Generate VMs
^^^^^^^^^^^^

Create the necessary number of virtual machines for an installation of SWITCHdrive. You will need machines for the following roles:

   * Loadbalancer (2 VCPU, 4 GB RAM, 20 GB Disk, Public IP Address)
   * Webserver (8 VCPU, 16 GB RAM, 20 GB Disk)
   * DB server (8 VCPU, 32 GB RAM, 40 GB Disk)
   * DB slave server (8 VCPU, 32 GB RAM, 40 GB Disk)
   * LDAP server (2 VCPU, 4 GB RAM, 20 GB Disk)
   * NFS server (2 VCPU, 4 GB RAM, 20 GB Disk)
   * Cloud ID server (1 VCPU, 2 GB RAM, 20 GB Disk)
   * Monitoring server (2 VCPU, 4 GB RAM, 20 GB Disk, Public IP Address)

Boot each VM and run through installation of OS. Make a note of the internal IP addresses of the VMs.

Create the necessary volumes:

   * data volume  (/dev/vdb on NFS server)
   * backup volume (/dev/vdc on NFS server)
   * log volume (/dev/vdb on syslog server)
   * db volume (/dev/vdb on db server)

Configuration of Environment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create an **environment** file in the ansible directory. Here is an example of
the **staging** file::

    [db]
    10.0.20.45 ansible_ssh_user=ubuntu

    [nfs]
    10.0.20.61 ansible_ssh_user=ubuntu

    [web]
    10.0.20.3 ansible_ssh_user=ubuntu

    [lb]
    10.0.20.63 ansible_ssh_user=ubuntu

    [ldap]
    10.0.20.62 ansible_ssh_user=ubuntu

    [monitoring]
    10.0.20.64 ansible_ssh_user=ubuntu

    [syslog]
    10.0.20.67 ansible_ssh_user=ubuntu

    [dev]

    [cmd]
    localhost ansible_connection=local

    [staging:children]
    db
    nfs
    web
    lb
    ldap
    monitoring
    syslog
    cmd

    [staging:vars]
    service_name=drive-stage.switch.ch
    ldap_ip=10.0.20.62
    ldap_host=stage-ldap
    ldap_password=ldap_secret_password
    nfs_ip=10.0.20.61
    db_ip=10.0.20.45
    syslog_ip=10.0.20.67
    admin_pass=owncloud_admin_password
    OWNCLOUD_VERSION='6.0.2'

The first part tells ansible which virtual machine is in which group (the
groups are ``[db]``, ``[web]`` etc. Note that a group can have more than one
group (however, the setup only works for the web group having multiple servers.
All other groups should only have one server)

The ``[staging:children]`` group collects all the servers into one group, so
that the next section ``[staging:vars]``, the variables, are visible to all
configured servers.

Initial Installation on VMs
^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can either install VMs from stock Ubuntu 13.10 images or create a volume
first, and then boot the VM from the volume.

The current setup is a mix of image and volume based servers. The original
thought was to make the servers boot from volume in order not to lose data due
to ephemeral disks being deleted. However, I am now (March 2014) in the process
of switching the servers to being image based with the databases/persistent data
on volumes. This makes for much easier recreation of VMs should they fail.

Current status:

============ ========= ====================================
VM           Status    Notes                                  
============ ========= ====================================
ldap         Volume    LDAP db needs to be persistent 
nfs          Volume    Can be moved to image as no data is persistent 
lb           Volume    ditto 
web          Volume    ditto 
db           Image     database stored on separate volume 
syslog       Image     log data stored on separate volume 
zabbix       Volume    database needs to be moved to volume 
============ ========= ====================================

The following volumes need to be created:

  * owncloud data (20TB) - nfs  /dev/vdb
  * owncloud backups (20TB) - nfs /dev/vdc
  * sylog logs (100GB) - syslog /dev/vdb
  * db data (100GB) - db /dev/vdb

and attached to the correct servers.

The file systems on the syslog and db servers are created automatically by the
ansible playbook. In general we create the filesystem directly on the disk,
without partitioning it. This allows the volume to be resized without resizing a
partition on it, which makes the process simpler.

Due to historical reasons, the owncloud data lies on a partition (/dev/vdb1) so
you need to manually create that partition and format it::

    # on the NFS server
    parted /dev/vdb    # and then create a logical partition
    mkfs.xfs /dev/vdb1
    mkfs.xfs /dev/vdc

Manual configurations
^^^^^^^^^^^^^^^^^^^^^

You can (and should) edit the variables in the environment file (``staging``,
``production``).:::

    [staging:vars]
    # the dns entry name of the service (maps to the public IP of the loadbalancer)
    service_name=drive-stage.example.com

    # the internal IP address of the LDAP Server (same as the one above)
    ldap_ip=10.0.20.62

    # The host name of the LDAP server
    ldap_host=stage-ldap

    # The password that the cn=admin account has on the LDAP server
    ldap_password=secret_ldap_password

    # The internal IP address of the NFS server
    nfs_ip=10.0.20.61

    # The internal IP address of the database server
    db_ip=10.0.20.45

    # The internal IP address of the syslog server
    syslog_ip=10.0.20.67

    # The admin password for the owncloud instance (login is "admin")
    admin_pass=secret_password

    # The version number of the ownCloud clode installed
    OWNCLOUD_VERSION='6.0.2'

    # Is this an enterprise version of owncloud? Set to true, it will install
    # the enterprise version (the tgz file must be in 'roles/owncloud/files/owncloudEE)
    enterprise=true

    # is this a staging system? If true, the public keys in 'roles/common/files/dev_keys'
    # are added to the 'authorized_keys' of the ubuntu user of the virtual machines
    staging_system=true

    # The email address that is being used to send LDAP statistics to
    stats_send_to=owncloud-stats@example.com
    stats_from=owncloud-stats@example.com

    # server's mails to root (cronjobs) are sent to this address
    notification_mail=drive-operations@example.com

**Note** - yes it's not DRY to list the various IP addresses again, as they
could be computed from the hostsvars. However, for that to work, every single
webserver playbook has to reach out to all the hosts that it would need the IP
from. That makes the deployment of a single group of servers take longer. I have
chosen to duplicate those IP adresses instead...

SSH Certificates
^^^^^^^^^^^^^^^^

You will need a valid SSL certificate for the load balancer. The PEM file (with
CRT and KEY concatenated) needs to be stored in
``roles/haproxy/files/service_name.pem`` where ``service_name`` is the name of
the service as stored in the inventory file of Ansible (``production``,
``staging``)

The LDAP server has a self signed certificate. To create that, run the following
command::

  ansible-playbook -i staging -s prepare_ssl.yml

This will create and distribute the LDAP certificate to the correct positions in
the ansible directory structure.

Installation
^^^^^^^^^^^^

Once everything is configured, run the site ansible playbook::

    ansible-playbook -i staging -s site.yml

This will install the necessary software on all machines and configure them.

The Zabbix db server needs to be manually configured. Run these commands inside
the Zabbix VM::

  zcat /usr/share/zabbix-server-pgsql/{schema,images,data}.sql.gz | psql -h localhost zabbix zabbix

The actual Zabbix configuration needs to be done manually. In the directory
``roles/zabbix/files/`` there are three Zabbix exports that can be used as a
starting point to configure the server.

You will also need to grant access to two tables in the owncloud database:

On the db server::

  $ psql owncloud
  grant select on oc_ldap_user_mapping to zabbix;
  grant select  on oc_filecache to zabbix;

  grant select on oc_ldap_user_mapping to nagios;
  grant select  on oc_filecache to nagios;


.. links

.. _Git: http://git-scm.org
.. _Ansible: http://ansible.com
.. _`Ansible installation`: http://docs.ansible.com/intro_installation.html
