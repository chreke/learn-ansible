Learning Ansible
================

Set up Django + uWSGI + Nginx
-----------------------------

Ansible should do the following things:

 * Install required packages.

 * Install Python libs.

 * Configure the app so everything works together

 * Provide a local "dev" setup and a "live" setup

 * The "live" setup should use separate servers for the application
   and the database

Setup
-----

Install Ansible (>2.0) and Vagrant

Ansible + Vagrant
-----------------

We'll start with a `Vagrantfile` copied from the [Ansible Vagrant
docs](http://docs.ansible.com/ansible/guide_vagrant.html):

```ruby
# This guide is optimized for Vagrant 1.7 and above.
# Although versions 1.6.x should behave very similarly, it is recommended
# to upgrade instead of disabling the requirement below.
Vagrant.require_version ">= 1.7.0"

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/trusty64"

  # Disable the new default behavior introduced in Vagrant 1.7, to
  # ensure that all Vagrant machines will use the same SSH key pair.
  # See https://github.com/mitchellh/vagrant/issues/5005
  config.ssh.insert_key = false

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "v"
    ansible.playbook = "playbook.yml"
  end
end
```

Touch `playbook.yml` in the same directory as the `Vagrantfile` and
we're good to run `vagrant up`.

Install Python
--------------

Let's ensure that Python is installed:

```yaml
---
- hosts: default
  become: yes
  become_user: root
  tasks:
  - name: Install Python 3.4
    apt: name='{{ item }}' state=present
    with_items:
      - python3.4
      - python3.4-dev
      - python3-pip
```

For some reason the Python we got from APT refuses to install `pip`,
so we'll install `python3-pip` as well.

Set up port forwarding and synced folders
-----------------------------------------

Add these lines to the `Vagrantfile`:

```ruby
config.vm.network "forwarded_port", guest: 8080, host: 4000

config.vm.synced_folder "src/", "/home/vagrant/src"
```

Variables
---------

The next step is to add some variables containing information about
our app:

```yaml
vars:
  app_name: foo
  app_dir: /home/vagrant/src
```

Install Django
--------------

Add a `requirements.txt` with this line:

    Django==1.9

Then add the following lines to the playbook:

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

First we should add a variable to the playbook containing the location
of our Python executable:

```yaml
python_bin: '{{ app_dir }}/venv/bin/python'
```

Then we create a new template in `templates/django.conf.j2`:

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
a loopback interface and will not work with port forwarding.

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

Set up Postgres
---------------

The next step is to set up Postgres. By default, Django sets up
a SQLite database, but since I predict that my test app will become
a massive viral success (if you are a venture capitalist willing to
provide series A funding please get in touch) we need something more
performant.

Let's add the following variables:

```yaml
db_name: '{{ app_name }}'
db_user: '{{ app_name }}'
db_password: 'password'
```

(Feel free to substitute `'password'` with something more secure. Or
don't. #yolo)

Then add these tasks to the playbook - they should run *before* we
install Python dependencies:

```yaml
- name: Install PostgreSQL
  apt: name='{{ item }}' state=installed
  with_items:
    - postgresql
    - postgresql-contrib
    - python-psycopg2
- name: Ensure PostgreSQL is running
  service: name=postgresql state=started
- name: Ensure database is created
  become_user: postgres
  postgresql_db:
    name: '{{ db_name }}'
    encoding: UTF-8
    lc_collate: en_US.UTF-8
    lc_ctype: en_US.UTF-8
    template: template0
    state: present
- name: Ensure user has access to the database
  become_user: postgres
  postgresql_user:
    db: '{{ db_name }}'
    name: '{{ db_user }}'
    password: '{{ db_password }}'
    priv: ALL
    state: present
    role_attr_flags: NOSUPERUSER,NOCREATEDB
```

To clarify, `psycopg2` is required for Ansible to talk to Postgres, so
that's why we're installing it here.

Connecting Django and Postgres
------------------------------

My idea is that the Django app will get its configuration from
environment variables. This has the advantage of not requiring Ansible
to create Django settings files, which makes the app more portable.

To do this we will use the `dj-database-url` package. Add this line to
`requirements.txt`:

    dj-database-url==0.4

In `settings.py` we can replace the default database config with:

```python
import dj_database_url

DATABASES = {
    'default': dj_database_url.config(conn_max_age=600)
}
```

We also need to add `psycopg2` as a dependency in `requirements.txt`:

    psycopg2==2.6.1

Wait, did you just install `psycopg2`... again? Yes.

![](dealwithit.gif "deal with it")

`dj-database-url` expects to find a variable named `DATABASE_URL` in
the environment. To set it up we'll use the `environment` keyword. Put
this on the top level of the play:

```yaml
environment:
  DATABASE_URL: >-
    postgres://{{ db_user }}:{{ db_password }}@localhost/{{ db_name }}
```

(The funny `>-` sigil lets us write a single-line string over multiple
lines. This is almost the same as `>`, except the latter adds
a trailing newline. The more you know!)

Now we should be able to run Django's migrations. Lucky for us,
Ansible comes with a module for running Django admin commands, called
`django_manage`. Add the following step to the playbook, just before
starting Django:

```yaml
- name: Run migrations
  django_manage:
    command: migrate
    app_path: '{{ app_dir }}'
    virtualenv: '{{ app_dir }}/venv'
```

Using uWSGI + Nginx
-------------------

Let's use Nginx and uWSGI to serve our Django application. uWSGI will
handle requests to the app and Nginx will serve static files.

We'll start with installing Nginx:

```yaml
- name: Install Nginx
  apt: name=nginx state=installed
```

Next, we'll install uWSGI - note that we don't involve our
`virtualenv` here, since uWSGI doesn't need to know about our
dependencies:

```yaml
- name: Install uWSGI
  pip:
    name: uwsgi
    version: 2.0
    executable: pip3
```

We also need to configure the two. We'll start by rewriting the
`django.conf.j2` template to start uWSGI instead - change the `exec`
line to:

```
exec uwsgi \
    --chdir {{ app_dir }} \
    --socket :8001 \
    --module {{ app_name }}.wsgi \
    --home {{ app_dir }}/venv
```

Next, we need to create a `uwsgi_params` file which Nginx will use.
Put this in a `src/uwsgi_param` file:

```
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;

uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;

uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
```

Finally, we'll create a site config for Nginx in
`templates/nginx.conf.j2`:

```nginx
upstream django {
    server 127.0.0.1:8001;
}

server {
    listen      8080;
    server_name 0.0.0.0;
    charset     utf-8;

    client_max_body_size 75M;

    location /media  {
        alias {{ app_dir }}/media;
    }

    location /static {
        alias {{ app_dir }}/static;
    }

    location / {
        uwsgi_pass  django;
        include     {{ app_dir }}/uwsgi_params;
    }
}
```

We'll use the `template` module to put this in
`/etc/nginx/sites-enabled/django-uwsgi.conf`:

```yaml
- name: Create site config for Nginx
  template:
    src: templates/django-uwsgi.conf.j2
    dest: /etc/nginx/sites-enabled/django-uwsgi.conf
```

To get Nginx to serve static files we need Django to collect these
files into a directory. Add this line to `settings.py`:

```python
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
MEDIA_ROOT = os.path.join(BASE_DIR, "media/")
```

(`MEDIA_ROOT` is for user-uploaded files)

Before Django is started, we make sure that static files are collected:

```yaml
- name: Collect static files
  django_manage:
    command: collectstatic
    app_path: '{{ app_dir }}'
    virtualenv: '{{ app_dir }}/venv'
```

Finally, we make sure that Nginx is started:

```yaml
- name: Start Nginx
  service: name=nginx state=restarted
```

Further reading
---------------

http://docs.ansible.com/ansible/playbooks_best_practices.html
