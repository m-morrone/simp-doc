General Upgrade Instructions
============================

Incremental Updates
-------------------

For ``Y`` and ``Z`` SIMP changes, you should feel comfortable dropping the changes
directly into your test systems. The promotion cycle from test to production
should be short and painless.

For RPM-based systems, a simple ``yum update`` should suffice after adding the
necessary packages to your site ``yum`` repositories. If you are using ``r10k``
or Code Manager, you will need to work with the upstream Git repositories as
appropriate for your workflow.

.. NOTE::

   If you started with an ISO installation, an easy way to get your entire
   local SIMP distribution updated is to download the new SIMP ISO and run
   ``unpack_dvd </path/to/ISO>``.

.. IMPORTANT::

   Be sure to review any version-specific upgrade instructions prior to
   executing the incremental upgrade. Although this type of upgrade will
   not contain any breaking changes, there may be specific instructions
   that you should follow to facilitate the upgrade process.

Breaking Changes
----------------

If the ``X`` version number has changed then you should expect **major**
breaking changes to the way SIMP works. Please carefully read the Changelog and
the new User's Guide and do **not** deploy these changes directly on top of
your production environment.

.. IMPORTANT::

   Upgrading SIMP does **not** require re-kicking your clients, even if some
   core services move to the new Puppet node.  All software configurations can
   be updated in Puppet, as needed.

New Server Creation and Client Migration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The recommended method for upgrading breaking changes is to create a new Puppet
master and migrate your data and clients to it. This process follows the path
of least destruction; we will guide you through how to back up the existing
Puppet master, create a new server, and transfer your clients.

#. Set up a new Puppet master that will house your new SIMP environment.

   .. NOTE::

      You must ensure that this node can be reached by any client that is to be
      migrated. The new system will not interfere with your existing Puppet
      system unless you specifically configure it to do so.

   .. IMPORTANT::

      Do **NOT** destroy your old Puppet master until everything has been
      successfully migrated and is in production under the new server.

#. Consider vital services other than Puppet that are housed on your current
   Puppet master node (e.g. DNS, DHCP, LDAP, custom kickstart, YUM, NFS, etc.).
   You may choose to keep many of these services running on your old Puppet
   master node. Anything not preserved must be migrated to a new system.

Back up the Existing Puppet Master
""""""""""""""""""""""""""""""""""

Prior to any modifications to your infrastructure, we **highly** recommend
following :ref:`ug-howto-back-up-the-puppet-master`.

Create a New Server
"""""""""""""""""""

Obtain an `official SIMP ISO <https://download.simp-project.com/simp/ISO/>`_ or point your
server at the latest `YUM Repositories <https://packagecloud.io/simp-project>`_
and follow the :ref:`simp-installation-guide`.

Follow the :ref:`Client_Management` guide, and set up services as needed.
Remember, you can opt-out of any core services (DNS, DHCP, etc.)  you want your
clients or old Puppet master to run! If you want the new Puppet master to run
services the existing Puppet master ran, you may be able to use the backup of
the ``rsync`` directories from the old system.

.. WARNING::

   Do not blindly drop ``rsync`` (or other) materials from the old Puppet
   master onto the new one. This is a breaking version and the required
   structures for these components may have changed.

When you :ref:`ug-apply-certificates` you may wish to transfer client certs to
the new server.  If you are using the FakeCA and still wish to preserve the
certificates, follow the :ref:`ug-apply-certificates-official-certificates`
guidance, and treat the existing Puppet master as your 'proper CA'.

Promote the New Puppet Master and Transfer Your Clients
"""""""""""""""""""""""""""""""""""""""""""""""""""""""

Follow the :ref:`ug-howto-change-puppet-masters` guide to begin integration
of your new Puppet master into the existing environment.

.. NOTE::

   You should *always* start migration with a small number of
   **least critical** clients!

Retire the Old Puppet Server
""""""""""""""""""""""""""""

Once you have transferred the management of all your clients over to
the new Puppet master, you may safely retire the old Puppet master.
