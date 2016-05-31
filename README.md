Learning Ansible
================

Set up Django + uWSGI + Nginx
-----------------------------

Ansible should do the following things:

 * Install required packages.

 * Install Python libs.

 * Configure the app so everything works together

 * Open up ports for local development.

 * Mount the application files on the local file system.

 * Provide separate "live" and "dev" plays

Setup
-----

Install Ansible (>2.0) and Vagrant

Ansible + Vagrant
-----------------

Copied `Vagrantfile` from here: http://docs.ansible.com/ansible/guide\_vagrant.html

Touch `playbook.yml` in the same directory as the `Vagrantfile` and
we're good to run `vagrant up`.

Install Python
--------------

I wanted to use Python 3.5, and already I found myself running into
"interesting" problems - turns out 3.5 is not in the official `apt`
repo yet, so we have to add a new PPA.

This can be fixed by adding the "deadsnakes" PPA (someone on the
Internet told me to).

This is what we have so far:

```yaml
---
- hosts: default
  become: yes
  become_user: root
  tasks:
  - name: Add deadsnakes PPA
    apt_repository: repo='ppa:fkrull/deadsnakes' state=present
  - name: Update APT cache
    apt: update_cache=yes
  - name: Ensure Python 3.5 is installed
    apt: name=python3.5 state=present
```
