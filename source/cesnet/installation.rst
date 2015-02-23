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
------

Puppet client has to be installed on all nodes participating in the ownCloud cluster.
All packages needed could be obtained from the Puppetlabs_ repository.
You will need to install some Puppet version < 3.0, because the modules we use
haven't been tested with *Puppet 3.x* versions yet.

There are two modes available when deploying the Puppet -- **standalone** client or **master/agent**.
Our Puppet configuration is tested with the master/agent setup, but it should work fine even when using just
a standalone clients. You can read more about these two modes and how to deploy Puppet in the `Puppet installation guide`_.

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
deployed to all nodes.

Pacemaker
---------

The basic installation of Pacemaker HA manager on RHEL 6 system is not covered in this text and can be found elsewhere_. In this section, let us assume that fully functional Pacemaker is installed on at least three hosts with working STONITHd and all necessary dependencies like, e.g., filesystem resources. We also assume that all necessary RAs are have been installed as part of the Pacemaker installation and placed in /usr/lib/ocf/resource.d/. The only remaining RA is the one for controlling PgPool II. This RA needs to be written, or, more conveniently, `CESNET version_` can be downloaded.

All examples of Pacemaker configuration are meant to be used with the help of crmshell_. Service definition may be the following::

        primitive PSQL_OC pgsql \
        op monitor interval=60s timeout=30s on-fail=restart \
        op start interval=0 timeout=600s on-fail=restart requires=fencing \
        op stop interval=0 timeout=120s on-fail=fence \
        params pgdata="/some_path/pgsql/data/" pghost=IP_address monitor_password=password monitor_user=user pgdb=monitor \
        meta resource-stickiness=100 migration-threshold=10 target-role=Started

Special database monitor is used for the monitoring of the PostgreSQL database. We recommend keeping minimally one connection to the database unhanded by PgPool II so this monitor can use it.
Another example is definition of PgPool II service based on our RA::

        primitive pgpool-owncloud-postgres ocf:du:pgpool_ra.rhel \
        params pgpool_conf="/pgpool_inst_path/etc/pgpool/pgpool.conf" pgpool_pcp="/pgpool_inst_path/etc/pgpool/pcp.conf" logfile="/log_path/pgpool/pgpool.log" pgdata="/pgsql_data_path/pgsql/data/" pghost=IP_address monitor_password=password monitor_user=user pgdb=monitor pgport=port \
        meta resource-stickiness=10 migration-threshold=10 target-role=Started \
        op monitor interval=60s timeout=40s on-fail=restart \
        op start interval=0 timeout=60s on-fail=restart requires=fencing \
        op stop interval=0 timeout=60s on-fail=fence

All remaining necessary services are configured in the same manner. Suitable parameters of different RAs can be tested by direct running of those scripts. For example the database can be monitored by this command::

        OCF_ROOT=/usr/lib/ocf OCF_RESKEY_pgdata="/some_path/pgsql/data/" OCF_RESKEY_pghost=IP_address OCF_RESKEY_monitor_password="password" OCF_RESKEY_monitor_user=user OCF_RESKEY_pgdb=monitor /usr/lib/ocf/resource.d/heartbeat/pgsql monitor

Next all location, colocation, and order linkages must by specified. In order to achieve our configuration as described in architecture file next lines must be added into Pacemaker configuration::

        location l-PSQL_OC-fe4 PSQL_OC 600: fe4-priv
        location l-PSQL_OC-fe5 PSQL_OC 500: fe5-priv
        location l-owncloud-web-fe4 owncloud-web 500: fe4-priv
        location l-owncloud-web-fe5 owncloud-web 600: fe5-priv

Location statement determines on which nodes the service should start with some priorities.
Colocation is used to specify which services should be started together on the same host and which must be on different hosts. Each colocation has appropriate weight or inf and -inf are used for absolute meanings. Resolving of the colocation dependencies is performed from right to left.::

        colocation c-FS-services inf: ( PSQL_OC owncloud-web ) FS
        colocation c-PSQL_OC-IP inf: PSQL_OC PSQL-ip
        colocation c-owncloud_web-IP inf: owncloud-web owncloud-ip owncloud-ipv6 pgpool-owncloud-postgres

So the last rule means that owncloud-web is run where owncloud-ip is running and that is on the same node as owncloud-ipv6 and that is where pgpool-owncloud-postgres service is running.
The last parameter changes the order in which are services started and stopped.::

        order o-FS-services inf: FS ( PSQL_OC owncloud-web )
        order o-PSQL-Owncloud_web inf: PSQL_OC pgpool-owncloud-postgres owncloud-web
        order o-PSQL_OC-IP inf: PSQL-ip PSQL_OC
        order o-owncloud_web-IP inf: owncloud-ip owncloud-web
        order o-owncloud_web-IPv6 inf: owncloud-ipv6 owncloud-web

After successful configuration of all services, fine tuning of timeout values is necessary according to overall behaviour of the system. There are no general values of timeouts, we recommend to start with the ones from RA scripts. 

Setting up ownCloud
-------------------

In the next step, you will need to download and install ownCloud from the source archive.
Just follow the `Download ownCloud`_ and `Set permissions`_ sections of the
official installation guide. Just put it in a directory specified in the Puppet's
'nodes.pp' variable 'webdir'.

For the user SAML authentication to work properly, you need to fetch the 'user_saml' app
from the `owncloud/apps`_ GitHub repository. It already contains our fixes of
the 'user_saml' app. If you are interested in our modifications as described in
the :ref:`cesnet-modifications` chapter, you can use the
`cesnet/owncloud-apps`_ repository instead.

Then you create 'owncloud' DB table and user and go through the
standard ownCloud webinstall. When you are done with installation,
it is important to note the **instanceid** generated by ownCloud. You
can find it in the ownCloud's 'config.php'. It will be needed by the SimpleSAMLphp.

SimpleSAMLphp
^^^^^^^^^^^^^

Now you'll need to finish the configuration of an authentication backend
used by the 'user_saml' app. Most of the things should be already
put in place by Puppet, but you will need to have a look and modify
some files referenced by the 'apache2::simplesamlphp'
Puppet class. In 'config.php.erb', you will need to change the cookiename to
the 'instanceid' noted in the section before::

	'session.phpsession.cookiename'  => 'oc1234567',

In the 'authsources.php.erb', change the attributes in the 'default-sp' section
according to your environment::

	'default-sp' => array(
                'saml:SP',
                'entityID' => 'https://<%= @oc_hostname %>/saml/sp',
                'idp' => NULL,
                'privatekey' => '<%= @key_dir %>/<%= @oc_hostname %>.key',
                'certificate' => '<%= @cert_dir %>/<%= @oc_hostname %>.crt',
                // eduID.cz + hostel WAYFlet
                'discoURL' => 'https://ds.eduid.cz/wayf.php...'

The last thing needed is to specify sources of IdPs (Identity Providers) metadata.
This can be done in the 'config-metarefresh.php.erb' file::
                
	'eduidcz' => array(
		'cron'          => array('daily'),
		'sources'       => array(
			array(
				'src' => 'https://metadata.eduid.cz/...',
			),
		),
		'expireAfter' => 60*60*24*4, // Maximum 4 days cache time.
		'outputDir'   => 'metadata/eduidcz/',
	)

Metadata are then refreshed periodically by a cron job already installed by Puppet.
More generic information about setting your own SP in the SimpleSAMLphp could be found
in the official configuration guide_.

User_saml Configuration
^^^^^^^^^^^^^^^^^^^^^^^

The last step needed to get the user authentication running is to enable
the 'user_saml' app in the ownCloud and configure it properly.
This can be done in the web administration interface. You just
need to set a proper paths to SimpleSAMLphp and user's
attribute mapping from SimpleSAMLphp to ownCloud according to your environment.
You can test which attribute names SimpleSAML gives you about a user
on its testing page::

	https://your.domain.com/simplesamlphp/module.php/core/authenticate.php?as=default-sp

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
.. _guide: https://simplesamlphp.org/docs/stable/simplesamlphp-sp
