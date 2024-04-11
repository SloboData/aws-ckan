# Install CKAN on AWS Ubuntu

Before installing CKAN we need to install pyenv and install python 3.9.19 because 2.9 doesnt support py 3.10+

Follow steps to install from README file

### Install the CKAN required packages

``` bash
sudo apt-get install python3-dev postgresql libpq-dev python3-pip python3-venv git-core openjdk-8-jdk redis-server libffi-dev libncurses-dev libbz2-dev libreadline-dev libsqlite3-dev
liblzma-dev nginx
```

### Setting up the virtual environment

``` bash
pyenv install 3.9.19
pyenv virtualenv local
# Navigate to virtual env location
cd .pyenv/versions/3.9.19/envs/local/
```

### Install CKAN from source

``` bash
pip install -e 'git+https://github.com/ckan/ckan.git@ckan-2.9.11#egg=ckan[requirements]'
```

> **_NOTE:_** If the requirements dont install or throw an error, following commands should be executed

``` bash
cd .pyenv/versions/3.9.19/envs/local/src/ckan/
pip install -r requirement-setuptools.txt 
pip install -r requirements.txt
python setup.py install
```

### Optional: Setup the database

To create an user and the database:

``` bash
sudo -u postgres createuser -S -D -R -P ckan_default
sudo -u postgres createdb -O ckan_default ckan_default -E utf-8

```

### Optiona: Setup SOLR

To install SOLR we need to download it frist

``` bash
cd /opt
wget https://archive.apache.org/dist/lucene/solr/8.11.3/solr-8.11.3.tgz
tar xzf solr-8.11.3.tgz solr-8.11.3/bin/install_solr_service.sh --strip-components=2
bash ./install_solr_service.sh solr-8.11.3.tgz -f
```

When installation ends we need to setup SOLR core

```bash
sudo su solr
cd /opt/solr/bin
# Delete the core if exists
./solr delete -c ckan
# Create new ckan core config
./solr create -c ckan
exit
```

When the core is set, we need to link the schema to SOLR

``` bash
sudo su
cd /var/solr/data/ckan/conf
rm managed-schema
cp /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.solr8.xml schema.xml
chown solr:solr schema.xml
systemctl restart solr
```

### Continue with CKAN setup

After successfully setup of the database and SOLR, make CKAN configuration folder

``` bash
mkdir -p /etc/ckan/default
# Create soft link to who.ini
ln -s /home/ubuntu/.pyenv/versions/3.9.19/envs/local/src/ckan/who.ini /etc/ckan/default/who.ini
```

Navigate to CKAN environment folder

``` bash
cd .pyenv/versions/3.9.19/envs/local/bin
# Generate ckan.ini 
./ckan generate config /etc/ckan/default/ckan.ini
# ... here you will need to set the values for SOLR / DB / ckan site_url
...
# Initialize the Database
./ckan -c /etc/ckan/default/ckan.ini db init
```

Runnung `./ckan -c /etc/ckan/default/clkan.ini run` would start CKAN

To continue with Prodcution setup, we need to configure NGINX and Supervisor

### Setup NGINX

Nginx configuration:

``` bash
cd /etc/nginx/sites-available
```

Here add new file ckan with following content:

``` conf
proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=cache:30m max_size=250m;
proxy_temp_path /tmp/nginx_proxy 1 2;

server {
    client_max_body_size 100M;
    location / {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
        proxy_cache cache;
        proxy_cache_bypass $cookie_auth_tkt;
        proxy_no_cache $cookie_auth_tkt;
        proxy_cache_valid 30m;
        proxy_cache_key $host$scheme$proxy_host$request_uri;
        # In emergency comment out line to force caching
        # proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
    }

}
```

To include this configuration we need to create soft link to sites-enabled

``` bash
ln -s /etc/nginx/sites-available/ckan /etc/nginx/sites-enabled/ckan
```

### Setup Supervisor

Supervisor will be used to manage the running of CKAN and workers

Navigate to supervsior configuration

``` bash
cd /etc/supervisor/conf.d/
# Create ckan-uwsgi.conf and ckan-jobworker.conf
touch ckan-wsgi.conf
touch ckan-jobworker.conf
```

For `ckan-wsgi.conf` add follwoing content

``` conf
[program:ckan-uwsgi]

command=/home/ubuntu/.pyenv/versions/3.9.19/envs/local/bin/uwsgi -i /etc/ckan/default/ckan-uwsgi.ini

; Start just a single worker. Increase this number if you have many or
; particularly long running background jobs.
numprocs=1
process_name=%(program_name)s-%(process_num)02d

; Log files - change this to point to the existing CKAN log files
stdout_logfile=/etc/ckan/default/uwsgi.OUT
stderr_logfile=/etc/ckan/default/uwsgi.ERR

; Make sure that the worker is started on system start and automatically
; restarted if it crashes unexpectedly.
autostart=true
autorestart=true

; Number of seconds the process has to run before it is considered to have
; started successfully.
startsecs=10

; Need to wait for currently executing tasks to finish at shutdown.
; Increase this if you have very long running tasks.
stopwaitsecs = 600

; Required for uWSGI as it does not obey SIGTERM.
stopsignal=QUIT
```

For the `ckan-jobworker.conf` we need to set following content

```conf
; =======================================================
; Supervisor configuration for CKAN background job worker
; =======================================================

; 1. Copy this file to /etc/supervisor/conf.d
; 2. Make sure the paths below match your setup


[program:ckan-jobworker]

; Use the full paths to the virtualenv and your configuration file here.
command=/home/ubuntu/.pyenv/versions/3.9.19/envs/local/bin/ckan -c /etc/ckan/default/ckan.ini jobs worker

; User the worker runs as.
user=www-data


; Start just a single worker. Increase this number if you have many or
; particularly long running background jobs.
numprocs=1
process_name=%(program_name)s-%(process_num)02d


; Log files.
stdout_logfile=/var/log/ckan/ckan-jobworker.stdout.log
stderr_logfile=/var/log/ckan/ckan-jobworker.stderr.log


; Make sure that the worker is started on system start and automatically
; restarted if it crashes unexpectedly.
autostart=true
autorestart=true


; Number of seconds the process has to run before it is considered to have
; started successfully.
startsecs=10

; Need to wait for currently executing tasks to finish at shutdown.
; Increase this if you have very long running tasks.
stopwaitsecs = 600
```

Additionally we need to create `wsgy.py` and `ckan-uwsgi.ini` files in `/etc/ckan/default` location

``` bash
cd /etc/ckan/default
touch wsgy.py
touch ckan-uwsgi.ini
```

For `wsgy.py` we need to set the content:

``` python
# -*- coding: utf-8 -*-
import os
from ckan.config.middleware import make_app
from ckan.cli import CKANConfigLoader
from logging.config import fileConfig as loggingFileConfig
config_filepath = os.path.join(
    os.path.dirname(os.path.abspath(__file__)), u'ckan.ini')
abspath = os.path.join(os.path.dirname(os.path.abspath(__file__)))
loggingFileConfig(config_filepath)
config = CKANConfigLoader(config_filepath).get_config()
application = make_app(config)
```

And for the `ckan-uwsgi.ini` we need to set:

``` ini
# -*- coding: utf-8 -*-
[uwsgi]
```

To reload all use

``` bash
sudo systemctl restart supervisor.service
sudo supervisorctl load
```

If everything is OK, we should see:

``` terminal
ckan-uwsgi:ckan-uwsgi-00         RUNNING   pid 1366, uptime 2:46:58
ckan-jobworker:ckan-jobworker-00       RUNNING   pid 1367, uptime 2:46:58
```