HOWTO Configure NFS
===================

.. contents:: This chapter describes multiple configurations of NFS including:
  :local:

All implementations are based on ``pupmod-simp-nfs``, ``pupmod-simp-simp_nfs``,
and ``pupmod-simp-simp``.

.. NOTE::

  ``pupmod-simp-simp_nfs`` and ``pupmod-simp-nfs`` are not core modules, and
  may need to be installed prior to following this guide.

Known Issues
------------

Stunnel and Autofs
^^^^^^^^^^^^^^^^^^
The ``autofs`` packages that were released with CentOS 6.8 (`autofs-5.0.5-122`_)
and CentOS 7.3 (`autofs-5.0.7-56`_) worked properly over a stunnel connection.

The release shipped with CentOS 6.9 (**5.0.5-132**)  and with CentOS 7.4 (**5.0.7-69**)
prevents any connection from happening to the local stunnel process and breaks mounts to
remote systems over stunnel connections.

To use NFS over stunnel and automount directories the old package must be used.
To determine what version of autofs is installed, run ``automount -V``.

To force the package to the version wanted despite the fact that a newer version is available:

First make sure the package is available via your package-management facility then
set the package version in Hiera:

In CentOS 7.4:

.. code-block:: puppet

  ---
  autofs::autofs_package_ensure:  '5.0.7-56.el7'

In Centos 6.9

.. code-block:: puppet

  ---
  autofs::autofs_package_ensure:  '5.0.5-122.el6'


This problem has been identified as bugs in autofs and are being publicly
tracked.

- CentOS 6.9  https://bugs.centos.org/view.php?id=13575.
- CentOS 7.4  https://bugs.centos.org/view.php?id=14080.

If you have any further questions about this please contact the SIMP Team.

Autofs Option in ``nfs::client::mount``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The autofs option in ``nfs::client::mount`` resource currently only works with 
indirect wild-card mounts.  For all other autofs options use the autofs module
directly.

SIMP-2944 in `JIRA Bug Tracking`_.

Kerberos and Home Directories
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The kerberos module is not fully integrated with home directories at this time.

SIMP-1407 in `JIRA Bug Tracking`_.

Exporting Arbitrary Directories
-------------------------------

**Goal:** Export ``/var/nfs_share`` on the server, mount as ``/mnt/nfs`` on the
client.

.. NOTE::

   If anything in this section does not make sense, there is a full working
   example of how to export NFS home directories in the ``simp_nfs`` module.

Server
^^^^^^

In ``site/manifests/nfs_server.pp``:

.. code-block:: puppet

  class site::nfs_server (
    Stdlib::AbsolutePath                             $data_dir     = '/var/nfs_share',
    Simplib::Netlist                                 $trusted_nets = simplib::lookup('simp_options::trusted_nets', { 'default_value' => ['127.0.0.1'] }),
    Array[Enum['none','sys','krb5','krb5i','krb5p']] $sec          = ['sys']
  ){
    include '::nfs::server'

    file { $data_dir:
      ensure => 'directory',
      owner  => 'root',
      group  => 'root',
      mode   => '0644'
    }

    if !$::nfs::stunnel {
      nfs::server::export { 'nfs_share':
        clients     => $trusted_nets,
        export_path => $data_dir,
        sec         => $sec,
        require     => File[$data_dir]
      }
    }
    else {
      # Stunnel needs to point at the local host
      nfs::server::export { 'nfs_share':
        clients     => ['127.0.0.1'],
        export_path => $data_dir,
        sec         => $sec,
        require     => File[$data_dir]
      }
    }
  }

In ``hosts/<your_server_fqdn>.yaml``:

.. code-block:: puppet

  nfs::is_server: true

  classes:
    - 'site::nfs_server'

Client
^^^^^^


In ``site/manifests/nfs_client.pp``:

.. code-block:: puppet

   class site::nfs_client (
    Simplib::Host                                    $nfs_server,
    Enum['none','sys','krb5','krb5i','krb5p']        $sec = 'sys'
  ){

     $_mnt_point = '/mnt/nfs'

     file { "${_mnt_point}":
       ensure => 'directory',
       mode   => '755',
       owner  => 'root',
       group  => 'root'
     }

     nfs::client::mount { "${_mnt_point}":
       nfs_server  => $nfs_server,
       remote_path => '/var/nfs_share',
       sec         => $sec,
       at_boot     => true,
       autofs      => false,
       require     => File["${_mnt_point}"]
     }
  }

In ``hosts/<your_client_fqdn>.yaml``:

.. code-block:: yaml

  nfs::is_server: false
  site::nfs_client::nfs_server: <your nfs server>

  classes:
    - 'site::nfs_client'

.. WARNING::

   Non-wildcard indirect autofs mounts configured via ``nfs::client::mount``
   are not working properly at this time. See SIMP-2944 in our
   `JIRA Bug Tracking`_.  You may wish to manually configure the mount via
   ``autofs::map::master``, and ``autofs::map::entry`` instead.

.. NOTE::

   The ``simp_nfs`` module contains a further example that includes the use of
   a NFS root on the server and indirect autofs with wildcards on the client.


Exporting Home Directories
--------------------------

**Goal:** Export home directories for LDAP users.

Utilize the SIMP profile module ``simp_nfs``:

  #. ``simp_nfs``: Manages client and server configurations for managing NFS
     home directories.
  #. ``simp_nfs::create_home_dirs``: Optional hourly cron job that binds to a
     :term:`LDAP` server, ``simp_options::ldap::uri`` by default, and creates a
     NFS home directory for all users in the LDAP server. Also expires any home
     directories for users that no longer exist in LDAP.

.. NOTE::

   The NFS daemon may take time to reload after module application.  If your
   users do not have home directories immediately after application or it takes
   a while to log in, do not panic!

.. NOTE::

   Any users logged onto a host at the time of module application will not have
   their home directories re-mounted until they log out and log back in.

default.yaml
^^^^^^^^^^^^

.. code-block:: yaml

  nfs::is_server: false
  simp_nfs::home_dir_server: <your nfs server>

  classes:
    - simp_nfs

Server
^^^^^^

.. code-block:: yaml

  nfs::is_server: true
  simp_nfs::export_home::create_home_dirs: true

  classes:
    - simp_nfs::export::home


Enabling/Disabling Stunnel
--------------------------

Stunnel is a means to encrypt your NFS data.

Enable
^^^^^^

If ``simp_options::stunnel`` is set to ``true``, you need only specify the
following, in the server's :term:`YAML` file:

.. NOTE::

  The following is set to prevent a cyclical connection of stunnel to itself,
  in the event the server is a client of itself.

.. code-block:: yaml

  nfs::client::stunnel::nfs_server: <your nfs server>

If ``simp_options::stunnel`` is set to ``false`` and you do not wish to globally
enable stunnel, you will also need to set the following, in default.yaml:

.. code-block:: yaml

  nfs::stunnel: true

Disable
^^^^^^^

If ``simp_options::stunnel`` is set to ``true``, but you do not want your NFS
traffic to go through stunnel, set the following, in default.yaml:

.. code-block:: yaml

  nfs::stunnel: false

If ``simp_options::stunnel`` is set to ``false`` then stunnel is already disabled.

Enabling Kerberos
-----------------

.. WARNING::

  This functionality is incomplete. It does not work with home directories.
  See ticket SIMP-1407 in our `JIRA Bug Tracking`_ .

In addition to the sharing code (not the stunnel code) above, add the following:

default.yaml
^^^^^^^^^^^^

.. code-block:: yaml

  classes:
    - 'krb5::keytab'

  nfs::secure_nfs: true
  simp_options::krb5: true

  krb5::kdc::auto_keytabs::global_services:
    - 'nfs'

Server
^^^^^^

.. code-block:: yaml

  classes:
    - 'krb5::kdc'

Clients
^^^^^^^

.. code-block:: yaml

  nfs::is_server: false

  classes:
    - 'simp_nfs'

.. _autofs-5.0.5-122: https://vault.centos.org/6.8/os/x86_64/Packages/autofs-5.0.5-122.el6.x86_64.rpm
.. _autofs-5.0.7-56: https://vault.centos.org/7.3.1611/os/x86_64/Packages/autofs-5.0.7-56.el7.x86_64.rpm
.. _JIRA Bug Tracking: https://simp-project.atlassian.net/
