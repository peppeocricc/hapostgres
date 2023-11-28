HA postgresql
==========
Ansible playbooks to build an HA postgres on docker environment from scratch.

Description of the infrastructure
---------------------------------

We build 3 server postgresql in a docker containers, let’s say ``db0``, ``db1`` and ``db2``. ``db0`` is the primary server,
``db1`` is a synchronous replica and ``db2`` is an asynchronous replica. 

The replications is done with repmgr.

Moreover we have 3 servers pgpool (also in docker containers) let’s say ``pgp0``, ``pgp1`` and ``pgp2``. 
When the three pgpool starts they decide together to assign a virtual IP to one of them, let’s say is ``pgp0``. 
``pgp0`` becomes the frontend server. 

The applications should never connect directly to the postgresql servers, they should use the virtual IP. 

``pgp0`` sends the write requests to ``db0`` and load balance the SELECT requests to the three postgresql servers.

In case of failure of ``db1`` or ``db2``, ``pgp0`` detects the failure and marks its state as down. Nothing is
done on the failed server, but ``pgp0`` won’t use it anymore to load balance SELECT requests.

In case of failure of ``db0``, the primary postgresql server, ``pgp0`` detects the failure, marks its state as
down, connects to ``db0`` and stops the postgresql server (in case it is not already down), then connects to
``db1`` and uses repmgr to promote it as the new primary. repmgr will also connect to ``db2`` and make it
stream from ``db1`` .

In case of failure of ``pgp0``, the three pgpool servers decide with a quorum mechanism to assign the
virtual IP to another pgpool server that becomes the new frontend server.

How to use the playbooks
------------------------

These playbooks were written for ansible 2.9.
We need some few modules that we can install with these commands::

  $ ansible-galaxy collection install ansible.posix
  $ ansible-galaxy collection install community.crypto
  $ ansible-galaxy collection install community.postgresql

Before running a playbook you need to write down the right IP addresses in the hosts file 
and check the variable file.

For the sake of simplicity there is only one variable file for all the nodes (``inventory\group_vars\all``).

Check the data directory, the ports, check that ``primary_db`` points to the primary database, check that
``virtual_IP`` and ``virtual_IP_interface`` are correct, then you are good to go.

This file is the only one that contains sensitive data, like passwords or ssh keys. You should change them.

Installing and configuring pgpool instances with docker_pgpool.yml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This playbook install docker, build the image, start the container and then 
install and configure a pgpool server in it.

``docker_pgpool.yml`` is idempotent. You’ll use the same playbook to install a
server from scratch and to update the configuration.

You can install or configure all the pgpool instances at the same time::

   $ ansible-playbook --extra-vars "node=all_pgpool" docker_pgpool.yml

This playbook doesn’t restart the server, as we don’t want to interrupt the service, but it will reload
it, and it will start it if it is in stopped state.

Installing and initializing a primary postgres instance with docker_init primary.yml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This playbook install docker, build the image, start the container, then install a postgresql server from scratch in it, initialize the database and register it as primary.

It fails if the data directory isn’t empty. 

You'll run it with::

  $ ansible-playbook --extra-vars "node=db0" docker_init_primary.yml

This script is not idempotent. If it fails for some reason (i.e. because the ``postgresql_port`` was taken)
before running it again check whether if it started the postgresql server and stop it, remove the data
directory just created, solve the issue that made the playbook fail, then run it again.

Installing and cloning a postgres replica instance with docker_init_replica.yml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This playbook install docker, build the image, start the container, then install a postgresql server from scratch in it, clone the database from the primary (indicated by the variable ``primary_db``) and register the server as stand-by. 

It fails if the data directory isn’t empty. 

You'll run it with::
  
  $ ansible-playbook --extra-vars "node=db1" docker_init_replica.yml

This playbook isn’t idempotent too, so the same considerations of the previous playbook apply.

Configuring postgresql instances
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For that purposes we’ll use the playbook ``docker_db_config.yml``. 

This playbook expects to find a working installation of postgresql in the data directory. 

It will fail otherwise.
It’s idempotent, you can run it as many times as you want.

Just run::

  $ ansible-playbook --extra-vars "node=all_db" docker_db_config.yml
