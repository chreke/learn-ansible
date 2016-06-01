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

Set up port forwarding and synced folders
-----------------------------------------

Add these lines to the `Vagrantfile`:

```ruby
config.vm.network "forwarded_port", guest: 8080, host: 4000

config.vm.synced_folder "src/", "/home/vagrant/src"
```

Install Django
--------------

Add a `requirements.txt` with the line `Django==1.9`, then add the
following;

```yaml
- name: Install virtualenv
  apt: name=python-virtualenv state=present
- name: Install Python dependencies
  pip:
    requirements: '{{ app_dir }}/requirements.txt'
    virtualenv_python: /usr/bin/python3.5
    virtualenv: '{{ app_dir }}/venv'
```

Running Django
--------------

The next step is to run Django. Just to see if everything works
I decided to test running the Django dev server. However, just
invoking `manage.py runserver` is not really a good idea - the problem
is that Ansible is supposed to be a declarative language, and so every
command in the playbook should be idempotent.

Idempotency means that performing an action multiple times will not
affect the outcome, which is not true for `runserver` - if the server
is already running when we attempt to start it, it will fail with an
error because the port is already in use.

Fortunately, we can use Debian's Upstart service to conditionally
start applications, but that means we have to write an upstart conf
for Django. On to templating!

I added a `python\_bin: '{{ app\_dir }}/venv/bin/python'` variable
which I used in this template:

```
description "Start Django app"
start on runlevel [2345]
stop on runlevel [06]
respawn
respawn limit 10 5
exec {{ python_bin }} {{ app_dir }}/manage.py runserver 0.0.0.0:8080
```

Note that we have to start the server on `0.0.0.0` - this is actually
different from `127.0.0.1` (Django's default) since the latter is
actually a loopback interface and not the machine's own IP address.

We then use Ansible's `template` module to create the upstart conf
file:

```yaml
- name: Create upstart config for Django
  template: src=templates/django.conf.j2 dest=/etc/init/django.conf
```

I'm a bit unsure about the best way to structure the templates
directory - maybe it should mirror where the templated files should
live on the guest machine's file system?

Anyway, now that we have defined our `django` service we can start it
like so:

```yaml
- name: Start Django
  service: name=django state=restarted
```

Just to make sure that `upstart` picks up any config changes I restart
the service every time. (Maybe it would be better to use `when` here?)
