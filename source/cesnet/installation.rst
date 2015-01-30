CESNET Installation
===================

Prerequisites
-------------

In order to replicate our ownCloud deployment, you will need the following software
prerequisites to be installed on your designated nodes:

  * Puppet 2.7.x
  * Pacemaker
  * Munin-node
  * GPFS
  * Nrpe + nagios-plugins-nrpe
  * RHEL6 or Debian

For monitoring and reporting purposes, you should also have some dedicated servers
at your disposal with Nagios (Icinga) and Munin servers installed.

Preparing storage
-----------------

You should start by deciding where to place the whole ownCloud installation.

We have decided to place everything ownCloud related on a shared filesystem mounted on all nodes.
This allows achieving High Availability, Pacemaker is then able to move services between the nodes freely if one of the
nodes fails.
GPFS shared filesystem creation and mounting is described on the `IBM wiki`_.
We created a filesystem named *owncloud* using the following stanza file::

  # owncloudfs.stanza
  # This assumes you have already created NSDs
  # (mmaddisk) from your disk array LUNs
  %nsd:
          nsd=name_of_nsd
          usage=dataAndMetadata
  ...

And then executing the following commands (this will assign NSDs to the newly created filesystem)::
  
   mmcrfs owncloud -F owncloudfs.stanza -A no -B 2M -D nfs4 -n 5
   mmmount owncloud -a

You should now have the filesystem prepared and mounted on all nodes in
the GPFS cluster. 

Puppet
^^^^^^

Puppet client has to be installed on all nodes participating in the ownCloud cluster.
All packages needed could be obtained from the Puppetlabs_ repository.
You will need to install some Puppet version < 3.0, because the modules we use
aren't tested with *Puppet 3.x* versions yet.

There are two modes available when deploying the Puppet -- **standalone** client or **master/agent**.
Our Puppet configuration is tested with the master/agent setup, but it should work fine even when using just a standalone clients. You can read more about these two modes and how to deploy Puppet in the `Puppet installation guide`_.

With Puppet installed and properly configured, download the following puppet modules:

Apache module::

  https://github.com/CESNET/puppet-apache #(to be uploaded)

PostgreSQL module (including PgPool II)::

  https://github.com/CESNET/puppet-postgres #(to be uploaded)

Put these two modules into the 'modulepath' (``/etc/puppet/modules/`` by default) of Puppet.
Both modules are then linked together and initialized by an ownCloud profile inside the
'nodes.pp' manifest. We then assign the profile to the ownCloud cluster nodes. 

.. NOTE::
  We use Puppet's Hiera_ for storing the variables, but you can
  avoid using Hiera by substituting all ``hiera()`` calls with variable values.

The relevant part of our 'nodes.pp' manifest follows::

  #/etc/puppet/manifests/nodes.pp
  class profile::owncloud (
    $base_dir  = '',
    $log_dir   = hiera('owncloud::log_dir')
  ) {

    # Configure PostgreSQL services
    # -----------------------------
    include ::postgres::backup
    
    class { '::postgres::server':
      conf_dir   => hiera('postgres::conf_dir', "${base_dir}/pgdata/"),
      listen_ip  => hiera('postgres::listen_ip')
    }

    class { '::postgres::pgpool':
      conf_dir         => hiera('postgres::pgpool::conf_dir',
                                "${base_dir}/etc/pgpool"),
      backend_hostname => hiera('postgres::listen_ip'),
    }

    # Configure Apache services
    # -------------------------
    $modpkgs = ['mod_ssl','mod_xsendfile']
    $config  = 'apache2/etc/httpd/httpd_oc.conf.erb'

    class { '::apache2::server':
      base_dir        => $base_dir,
      httpd_source    => $config,
      enabled_modules => ['ssl', 'xsendfile', 'rewrite'],
      disabled_sites  => ['default', 'default-ssl'],
      module_pkgs     => $modpkgs,
      manage_service  => true,
      reload_cmd      => $reloadcmd,
      oldlogs_dir     => "${log_dir}/old-logs/"
    }

    class {'::apache2::php':
      extension_packages  => [
        'php54', 'php54-php',
        'php54-php-cli', 'php54-php-common', 'php54-php-devel',
        'php54-php-gd', 'php54-php-mbstring', 'php54-php-pdo',
        'php54-php-pear', 'php54-php-pgsql',
        'php54-php-process', 'php54-php-xml', 'php54-runtime',
      ],
      php_module          => 'modules/libphp54-php5.so',
      post_max_size       => '16G',
      upload_tmp_dir      => "${base_dir}/tmp",
      upload_max_filesize => '16G',
    }

    include ::apache2::simplesamlphp
    
    class { '::apache2::owncloud': webdir => hiera('owncloud::webdir') }
  }

  node /your-node.hostnames.com/ {
    class { 'profile::owncloud': base_dir => '/yours/gpfs/mountpoint' }
  }

When using Puppet in a standalone mode, issue the following command on each node::

  # puppet apply /etc/puppet/manifests/nodes.pp

If you are running in the master/agent mode, deployment will be done automatically
by Puppet agents. This way you should now have all the ownCloud specific services
configured on all nodes.

Setting up Owncloud
-------------------

In the next step, you will need to download and install ownCloud 6 from the source archive.
Just follow the `Download ownCloud`_ and `Set permissions`_ sections of the
official installation guide.

For the user SAML authentication to work properly, you need to fetch the 'user_saml' app
from the `owncloud/apps`_ GitHub repository. It already contains our fixes of
the 'user_saml' app. If you are interested in our modifications as described in
the :ref:`cesnet-modifications` chapter, you are free to try the
`cesnet/owncloud-apps`_ repository instead.

SimplesamlPHP
^^^^^^^^^^^^^

Now you'll need to finish the configuration of an authentication backend
used by the 'user_saml' app. Most of the things should be already
put in place by Puppet.




Pacemaker
^^^^^^^^^

The basic installation of Pacemaker HA manager on RHEL 6 system is not goal of this text and can be find elsewhere_. For this section let's assume that fully functional installation of Pacemaker is installed on at least three hosts with working STINITHd and all necessary dependencies like filesystem resources and so on. Let's also assume that all necessary RAs are have been installed as part of the Pacemaker installation and placed in /usr/lib/ocf/resource.d/. Only missing RA is one for controlling PgPool II that needs to be written or `CESNET version_` can be downloaded.

All examples of Pacemaker configuration are meant to be used with the help of crmshell_ and service definition may looks like this::

        primitive PSQL_OC pgsql \
        op monitor interval=60s timeout=30s on-fail=restart \
        op start interval=0 timeout=600s on-fail=restart requires=fencing \
        op stop interval=0 timeout=120s on-fail=fence \
        params pgdata="/some_path/pgsql/data/" pghost=IP_address monitor_password=password monitor_user=user pgdb=monitor \
        meta resource-stickiness=100 migration-threshold=10 target-role=Started

Special database monitor is used for the monitoring of the PostgreSQL database. It's good to keep minimally one connection to the database unhanded by PgPool II so this monitor can use it.
Another example is definition of PgPool II service based on our RA::

        primitive pgpool-owncloud-postgres ocf:du:pgpool_ra.rhel \
        params pgpool_conf="/pgpool_inst_path/etc/pgpool/pgpool.conf" pgpool_pcp="/pgpool_inst_path/etc/pgpool/pcp.conf" logfile="/log_path/pgpool/pgpool.log" pgdata="/pgsql_data_path/pgsql/data/" pghost=IP_address monitor_password=password monitor_user=user pgdb=monitor pgport=port \
        meta resource-stickiness=10 migration-threshold=10 target-role=Started \
        op monitor interval=60s timeout=40s on-fail=restart \
        op start interval=0 timeout=60s on-fail=restart requires=fencing \
        op stop interval=0 timeout=60s on-fail=fence

All other services are configured in the same manner. Right parameters of different RAs can be tested by direct running of those scripts. For example the above database can be monitored by this command::

        OCF_ROOT=/usr/lib/ocf OCF_RESKEY_pgdata="/some_path/pgsql/data/" OCF_RESKEY_pghost=IP_address OCF_RESKEY_monitor_password="password" OCF_RESKEY_monitor_user=user OCF_RESKEY_pgdb=monitor /usr/lib/ocf/resource.d/heartbeat/pgsql monitor

Next all location, colocation and order linkages must by specified. 

After successful configuration of all services fine tuning of each of timeouts must take place. There is no general values of timeouts, but good start is the use of recommended ones from RA's scripts. 



TODO: we are changing our pacemaker configuration right now. This section
will be added when things get sorted out.

Configuration
-------------

ownCloud
^^^^^^^^

User_saml
^^^^^^^^^



.. links

.. _Git: http://git-scm.org
.. _Puppet: http://puppetlabs.com/
.. _Puppetlabs: http://docs.puppetlabs.com/guides/puppetlabs_package_repositories.html
.. _Hiera: http://docs.puppetlabs.com/hiera/1/
.. _`Puppet installation guide`: http://docs.puppetlabs.com/guides/install_puppet/pre_install.html#general-puppet-info
.. _`Puppet master`: http://docs.puppetlabs.com/guides/install_puppet/install_el.html#step-3-install-puppet-on-the-puppet-master-server
.. _`IBM wiki`: https://www.ibm.com/developerworks/community/wikis/home?lang=en#!/wiki/General+Parallel+File+System+%28GPFS%29/page/Install+and+configure+a+GPFS+cluster+on+AIX
.. _`Download ownCloud`: http://doc.owncloud.org/server/6.0/admin_manual/installation/installation_source.html#download-extract-and-copy-owncloud-to-your-web-server
.. _`Set permissions`: http://doc.owncloud.org/server/6.0/admin_manual/installation/installation_source.html#set-the-directory-permissions
.. _`owncloud/apps`: https://github.com/owncloud/apps
.. _`cesnet/owncloud-apps`: https://github.com/CESNET/owncloud-apps
.. _elsewhere: http://clusterlabs.org/quickstart-redhat.html
.. _crmshell: http://crmsh.github.io/
