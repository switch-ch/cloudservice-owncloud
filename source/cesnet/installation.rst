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

You should start by deciding, where to place the whole ownCloud installation.

We have decided to place everything ownCloud related on a shared filesystem mounted on all nodes.
This allows the Pacemaker to move services between the nodes freely if one of the
nodes fail, and thus to achieve High Availability.
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
We recommend to install *puppet-2.7.x* version package, because the modules we use
aren't tested with *Puppet 3.x* versions. 

There are two modes when running the Puppet -- **standalone** client and **master/agent**.
Our Puppet configuration is tested with master/agent setup, but it should work when using standalone clients
too. You can read more about these two modes and how to configure them on the `Puppet installation guide`_.

If you dont't want to use Hiera_ as an external data source, feel free to replace all occurences of the ``hiera()``
calls in our puppet manifests with variable values.

With puppet installed and properly configured, download the following puppet modules:

Apache specific configuration::

  https://github.com/CESNET/puppet-apache #(to be uploaded)

Postgres specific configuration::

  https://github.com/CESNET/puppet-postgres #(to be uploaded)

Put these two modules into the ``modulepath`` (``/etc/puppet/modules/`` is the default) of Puppet.
If you decided to use Hiera, you will need to create following files::

  #/etc/puppet/hiera.yaml
  ---
  :backends:
    - json

  :json:
    :datadir: /etc/puppet/hiera
    :hierarchy:
      - global

and::

  #/etc/puppet/hiera/global.json
  {
    "owncloud::log_dir" : "/gpfs/owncloud/logs",
    "owncloud::reload_cmd" : "/sbin/service/httpd reload"
  }


After that, you can prepare the PostgresSQL and Apache installation by putting the following content
into the ``/etc/puppet/manifests/nodes.pp``. You can replace ``hiera()`` calls with (default or custom) values,
if you are not using hiera with your Puppet installation. ::

  #/etc/puppet/manifests/nodes.pp
  class profile::owncloud (
    $base_dir  = '',
    $log_dir   = hiera('owncloud::log_dir', '/var/log'),
    $reloadcmd = hiera('owncloud::reload_cmd')
  ) {

    include postgres
    include postgres::backup
    class { '::postgres::pgpool':
      config_directory => "${base_dir}/etc/pgpool",
      backend_hostname => '<your.db.server.ip>',
                          # aka "postgres-ip" floating ip alias
    }

    case $::operatingsystem {
      'Debian': {
        $modpkgs = ['libapache2-mod-xsendfile']
      }
      'RedHat': {
        $modpkgs = ['mod_ssl','mod_xsendfile']
        $config  = 'apache2/etc/httpd/httpd_oc.conf.erb'
      }
      default: { fail("Owncloud is not supported on ${::operatingsystem}") }
    }
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
    class { '::apache2::simplesamlphp':
      authsources_source => 'apache2/etc/simplesamlphp/authsources-owncloud.php',
      authsources_path   => "${base_dir}/www/simplesamlphp/config/authsources.php",
      config_source      => 'apache2/etc/simplesamlphp/config-owncloud.php',
      config_path        => "${base_dir}/www/simplesamlphp/config/config.php",
    }
    class { '::apache2::owncloud':
      webdir => hiera('owncloud::webdir', '/var/www/owncloud')
    }
  }

  node /your-node.hostnames.com/ {
    class { 'profile::owncloud': base_dir => '/gpfs/owncloud' }
  }

.. NOTE::
      After the configuration of Pacemaker, you may want to change
      the ``$reloadcmd`` variable. If you want Puppet
      to instruct Pacemaker to reload the web service when configuration
      changes, you may set it to:
      ``/usr/sbin/crm resource restart owncloud-web``

      If you want to reload the service manually, just put
      ``/bin/true`` there and set ``manage_service => false`` for
      the ``::apache2::server`` class.

When using Puppet in a standalone mode, issue the following command on each node::

  # puppet apply /etc/puppet/manifests/nodes.pp

If you are running in master/agent mode, you can get yourself a cup of coffee while
the Puppet agents are fetching configuration from the Puppet master and doing its job.
You can however speed things up by running the following command on each node::

  # puppet agent --test

This will install and configure the Apache and PosgreSQL servers on all nodes
with matching hostnames for you. If you do not specify ``base_dir``, it will
write its configuration into default paths (mostly ``/etc/...``) for each package.
Because we use shared gpfs volume ``/gpfs/owncloud``, we tell Puppet to install
configuration into that volume (``/gpfs/owncloud/etc/...``).

Setting up Owncloud
-------------------

You will need to download and install ownCloud 6
from a source archive. Full installation procedure is described in
the `ownCloud installation guide`_, you can just skip the Apache & PostgreSQL
parts.

For the user authentication to work properly, you may want to check out our
`ownCloud apps`_ repository, especially the *user_saml* and *mail_notifications* app.
You can install these apps by running: ::

  cd /tmp/
  git clone https://github.com/CESNET/owncloud-apps
  cp -r owncloud-apps/user_saml /gpfs/owncloud/www/owncloud/apps/
  cp -r owncloud-apps/mail_notifications /gpfs/owncloud/www/owncloud/apps/

Now you should have ownCloud sources prepared, but you still need
to install and configure the Apache server together with PostgreSQL.
For this task we use Puppet.

SimplesamlPHP
^^^^^^^^^^^^^

You will also need to download the SimplesamlPHP authentication backend



Pacemaker
^^^^^^^^^

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
.. _`ownCloud installation guide`: http://doc.owncloud.org/server/6.0/admin_manual/installation/installation_source.html
.. _`ownCloud apps`: https://github.com/CESNET/owncloud-apps
