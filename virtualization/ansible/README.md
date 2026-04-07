# Ansible Lab – Role-Based Project Structure

## Context

In this lab, you will design and deploy a small infrastructure automation project using **Ansible**.

Ansible itself must be installed on a **separate administration machine**, inside a **Python virtual environment**. This machine will be used as the control node from which you will manage the target servers.

Your managed infrastructure consists of **four Debian Trixie servers**.

---

## General Requirements

You must write an Ansible project that applies a common baseline configuration to **all servers**.

On every server, your playbook must:

- install the following packages:
  - `rsyslog`
  - `screen`
  - `vim`
  - `sudo`
- create a user named `user`
- add `user` to the `sudo` group
- ensure that `sudo` is configured as a **secondary group** for `user`
- set the password of `user`
- set the password of `root`
- deploy the SSH public key of `user`

The passwords of `user` and `root` must be stored in the project using **Ansible Vault**, with values generated using `ansible-vault encrypt_string`.

---

## Server Roles

Among the four servers:

- one server must be configured as a **web server**
- one server must be configured as a **database server**
- the remaining two servers must stay **unassigned**

### Web Server

On the web server, you must:

- install `apache2`
- configure Apache so that:
  - `ServerSignature` is set to `Off`
  - `ServerTokens` is set to `Prod`

### Database Server

On the database server, you must:

- install `mariadb-server`
- configure a password for the database service

This database password must also be protected with **Ansible Vault**.

---

## Project Organization

Your project must be structured cleanly.

You must use:

- one inventory file named `infrastructure.yaml`
- `group_vars`
- `host_vars`

Your variable structure must include:

- one **common** variable file shared by all hosts
- one variable file for the **web server group**
- one variable file for the **database server group**

You should also organize the tasks into separate roles:

- `common`
- `web`
- `db`

These roles must be called from a top-level file named `main.yaml`.

---

## Project Tree

```text
ansible-lab/
├── ansible.cfg
├── infrastructure.yaml
├── playbook.yaml
├── group_vars/
│   ├── all.yaml
│   ├── webservers.yaml
│   └── dbservers.yaml
├── host_vars/
│   ├── www1.yaml
│   ├── db1.yaml
│   ├── node1.yaml
│   └── node2.yaml
└── roles/
    ├── common/
    │   └── tasks/
    │       └── main.yaml
    │   └── files/
    │       └── user_id_rsa.pub
    ├── web/
    │   ├── tasks/
    │   │   └── main.yaml
    │   ├── files/
    │   │   └── apache-security.conf
    └── db/
        └── tasks/
            └── main.yaml
```

---

## Top-Level Playbook

Create the file `playbook.yaml`:

```yaml
---
- hosts: all
  gather_facts: false
  tasks:
    - name: Apply common configuration to all hosts
      include_role:
        name: common
- hosts: webservers
  gather_facts: false
  tasks:
    - name: Configure web servers
      include_role:
        name: web
- hosts: dbservers
  gather_facts: false
  tasks:
    - name: Configure database servers
      include_role:
        name: db
```

Alternative, you can define two files `dbservers.yaml` and `webservers.yaml`:

```yaml
# dbservers.yaml
---
- hosts: dbservers
  gather_facts: false
  tasks:
    - name: Apply common configuration to all hosts
      include_role:
        name: common
    - name: Configure database servers
      include_role:
        name: db
```

```yaml
# webservers.yaml
---
- hosts: dbservers
  gather_facts: false
  tasks:
    - name: Apply common configuration to all hosts
      include_role:
        name: common
    - name: Configure web servers
      include_role:
        name: web
```


