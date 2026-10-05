AVChecker
=========

Avchecker is a simple tool to track availability of PostgreSQL in a cluster.
This role installs avchecker as a systemd service.
This role is part of PgVillage, which is an opinated PostgreSQL deployment for Virtual Machines.

Requirements
------------

None

Role Variables
--------------

Please see the [API documentation](docs/api.md) for a description of all variables.
The defaults are defined in [defaults/main.yml](defaults/main.yml).


Dependencies
------------

No dependencies


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - pgvillage.avchecker

License
-------

PostgreSQL

Author Information
------------------

PgVillage is an Open Community.
Main contributor is Nibble-IT.
