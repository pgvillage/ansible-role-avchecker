# pgvillage.avchecker API

This document describes all variables of the `pgvillage.avchecker` role.
Defaults are defined in [defaults/main.yml](../defaults/main.yml).

The role installs avchecker, a small Python script that tracks PostgreSQL
availability, as one or more `avchecker@<instance>` systemd services.
The role only does something when `avchecker_defaults` has at least one entry.

## Table of contents

- [General](#general)
  - [avchecker_user](#avchecker_user)
  - [avchecker_group](#avchecker_group)
  - [avchecker_path](#avchecker_path)
- [Packages](#packages)
  - [avchecker_package_state](#avchecker_package_state)
  - [avchecker_package_names](#avchecker_package_names)
  - [_avchecker_package_names](#_avchecker_package_names)
- [Instances](#instances)
  - [avchecker_defaults](#avchecker_defaults)
- [Certificates](#certificates)
  - [avchecker_certs_managed](#avchecker_certs_managed)
  - [avchecker_cert_folders](#avchecker_cert_folders)
  - [avchecker_root_cert](#avchecker_root_cert)
  - [avchecker_client_cert](#avchecker_client_cert)
  - [avchecker_client_key](#avchecker_client_key)
  - [avchecker_cert_files](#avchecker_cert_files)

## General

### avchecker_user

| Type   | Default     |
|--------|-------------|
| string | `avchecker` |

OS user that runs the `avchecker@` services. The role creates this user.
It is also created as a PostgreSQL role on instances where
`PGTARGETSESSIONATTRS` is `read-write`, and it is the default for `PGUSER` and
`PGDATABASE` when those are not set for an instance.

### avchecker_group

| Type   | Default     |
|--------|-------------|
| string | `avchecker` |

OS group of `avchecker_user`. The role creates it and uses it as the primary
group of `avchecker_user` and as the group of the systemd services.

### avchecker_path

| Type   | Default          |
|--------|------------------|
| string | `/opt/avchecker` |

Directory where the `avchecker.py` script is installed.

## Packages

### avchecker_package_state

| Type   | Default   |
|--------|-----------|
| string | `present` |

State passed to `ansible.builtin.package` for the prerequisite packages, for
example `present` or `latest`.

### avchecker_package_names

| Type | Default                                                      |
|------|--------------------------------------------------------------|
| list | Taken from `_avchecker_package_names` based on the package manager |

List of prerequisite packages to install. By default it is the entry in
`_avchecker_package_names` that matches the host's package manager
(`ansible_facts.pkg_mgr`), or the `default` entry if none matches.

### _avchecker_package_names

| Type       | Default |
|------------|---------|
| dictionary | See below |

Internal. Maps each package manager to the PostgreSQL Python driver
package(s) avchecker needs. To change the packages, override
`avchecker_package_names` instead.

```yaml
_avchecker_package_names:
  default:
    - python3-psycopg3
  apt:
    - python3-psycopg
  dnf:
    - python3-psycopg3
  zypper:
    - python313-psycopg2
```

## Instances

### avchecker_defaults

| Type       | Default |
|------------|---------|
| dictionary | `{}`    |

Dictionary of avchecker instances. Each key is an instance name. For each
key the role:

- writes the environment file `/etc/default/avchecker_<key>`
- enables and starts the `avchecker@<key>` systemd service

Each value is a dictionary of environment variables. When writing the service
environment file, keys are uppercased and `-` is replaced by `_`. PostgreSQL
provisioning uses the original keys, so use exact uppercase libpq names such as
`PGTARGETSESSIONATTRS`, `PGUSER` and `PGDATABASE`. Common variables:

| Variable               | Description                                                            |
|------------------------|------------------------------------------------------------------------|
| `PGHOST`, `PGPORT`     | libpq connection settings                                              |
| `PGUSER`               | Connection user. Defaults to `avchecker_user` for database creation     |
| `PGDATABASE`           | Database to use. Defaults to `${PGUSER}` for database creation       |
| `PGTARGETSESSIONATTRS` | When `read-write`, this instance is also used to create the PostgreSQL user, database and privileges |
| `AVCHECKER_SLEEPTIME`  | Seconds between checks (default `5`)                                   |

For `read-write` instances, the role creates the PostgreSQL user named `avchecker_user`.

This role creates all required users. The exact user(s) depend on the following:

- For each instance where PGUSER is set, that user is created and granted permissions as required.
- When PGUSER is not set, the role defaults to the setting for avchecker_user which defaults to the value avchecker.

The user will own the database (unless the postgres database is used) and have schema grants as required for avchecker to function properly.

Any other libpq environment variable (for example `PGSSLMODE`) can be set too.

When this dictionary is empty, which is the default, the role does nothing.

Example:

```yaml
avchecker_defaults:
  "5432":
    PGHOST: /tmp
    PGPORT: 5432
    PGTARGETSESSIONATTRS: read-write
```

## Certificates

### avchecker_certs_managed

| Type    | Default |
|---------|---------|
| boolean | `false` |

Whether the role deploys TLS certificates (`avchecker_cert_folders` and
`avchecker_cert_files`) for client certificate authentication to PostgreSQL.

### avchecker_cert_folders

| Type       | Default |
|------------|---------|
| dictionary | See below |

Dictionary of directories to create for certificates (mode `0700`) when
`avchecker_certs_managed` is `true`. Each value needs `path`, `owner` and
`group`. By default it is `~/.postgresql/` of `avchecker_user`, which is where
libpq looks for certificates by default.

```yaml
avchecker_cert_folders:
  postgres:
    path: "{{ getent_passwd[avchecker_user][4] }}/.postgresql/"
    owner: "{{ avchecker_user }}"
    group: "{{ avchecker_group }}"
```

### avchecker_root_cert

| Type   | Default          |
|--------|------------------|
| string | `---- CERT ----` |

PEM contents of the CA (chain) used to verify the PostgreSQL server
certificate. It is written to `root.crt`. Replace the placeholder when
`avchecker_certs_managed` is `true`.

### avchecker_client_cert

| Type   | Default          |
|--------|------------------|
| string | `---- CERT ----` |

PEM contents of the client certificate for `avchecker_user`. It is written
to `postgresql.crt`. Replace the placeholder when `avchecker_certs_managed` is
`true`.

### avchecker_client_key

| Type   | Default          |
|--------|------------------|
| string | `---- CERT ----` |

PEM contents of the client certificate's private key. It is written to
`postgresql.key`. Replace the placeholder when `avchecker_certs_managed` is
`true`. Consider storing it in Ansible Vault.

### avchecker_cert_files

| Type       | Default |
|------------|---------|
| dictionary | See below |

Dictionary of certificate files to deploy (mode `0600`) when
`avchecker_certs_managed` is `true`. Each value needs `path`, `body` (the file
contents), `owner` and `group`. Changes restart the `avchecker@` services.

```yaml
avchecker_cert_files:
  postgres_server_chain:
    path: "{{ avchecker_cert_folders.postgres.path }}/root.crt"
    body: "{{ avchecker_root_cert }}"
    owner: "{{ avchecker_user }}"
    group: "{{ avchecker_group }}"
  postgres_client_cert:
    path: "{{ avchecker_cert_folders.postgres.path }}/postgresql.crt"
    body: "{{ avchecker_client_cert }}"
    owner: "{{ avchecker_user }}"
    group: "{{ avchecker_group }}"
  postgres_client_key:
    path: "{{ avchecker_cert_folders.postgres.path }}/postgresql.key"
    body: "{{ avchecker_client_key }}"
    owner: "{{ avchecker_user }}"
    group: "{{ avchecker_group }}"
```
