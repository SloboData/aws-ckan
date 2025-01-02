# Installing CKAN 2.10 on AWS EC2 Ubuntu

AWS Ubuntu comes with preinstalled Python 3.10 which is supported by CKAN 2.10. So we can directly install CKAN 2.10 without installing pyenv or specific python version.

## Prerequisites

Before installing CKAN we need to install the following packages:

```bash
sudo apt-get update
sudo apt-get install python3-dev libpq-dev python3-pip python3-venv git-core redis-server
```

## Setting up the virtual environment and folder structure

```bash
sudo mkdir -p /usr/lib/ckan/default
sudo mkdir -p /etc/ckan/default
sudo chown `whoami` /usr/lib/ckan/default
sudo chown -R `whoami` /etc/ckan/
python3 -m venv /usr/lib/ckan/default
. /usr/lib/ckan/default/bin/activate
```

## Install CKAN from source

```bash
pip install --upgrade pip

pip install -e 'git+https://github.com/ckan/ckan.git#egg=ckan[requirements,dev]'
```

> **_NOTE:_** If the requirements don't install or throw an error, following commands should be executed

```bash
pip install -r /usr/lib/ckan/default/src/ckan/requirement-setuptools.txt
pip install -r /usr/lib/ckan/default/src/ckan/requirements.txt
```

### Optional: Setup the database

To create an user and the database:

``` bash
sudo -u postgres createuser -S -D -R -P ckan_default
sudo -u postgres createdb -O ckan_default ckan_default -E utf-8

```

### Optiona: Setup SOLR

To install SOLR we need to download it frist
> **_NOTE:_**  Use `sudo` if getting permission denied

``` bash
cd /opt
wget https://archive.apache.org/dist/lucene/solr/8.11.3/solr-8.11.3.tgz
tar xzf solr-8.11.3.tgz solr-8.11.3/bin/install_solr_service.sh --strip-components=2
bash ./install_solr_service.sh solr-8.11.3.tgz -f
```

When installation ends we need to setup SOLR core
> **_NOTE:_**  Use `sudo` if getting permission denied

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
cp /home/ubuntu/.pyenv/versions/3.9.19/envs/okgt/src/ckan/ckan/config/solr/schema.solr8.xml schema.xml
chown solr:solr schema.xml
systemctl restart solr
```

### Continue with CKAN setup

> **_NOTE:_**  Use `sudo` if getting permission denied

After successfully setup of the database and SOLR, make CKAN configuration folder

```bash
ckan generate config /etc/ckan/default/ckan.ini
```

Make a soft link to the configuration file

``` bash
sudo ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini
```

## Start CKAN

```bash
ckan -c /etc/ckan/default/ckan.ini run
```

To stop CKAN press `Ctrl + C`

## Setup NGINX

Navigate to the nginx configuration folder

```bash
cd /etc/nginx/sites-available
```

Here you need to add a new [configuration file](../nginx/ckan) for CKAN and then link it to the sites-enabled folder

```bash
sudo ln -s /etc/nginx/sites-available/ckan /etc/nginx/sites-enabled/ckan
```

After linking the configuration file, check the nginx config file and then restart the NGINX server

```bash
sudo nginx -t
sudo systemctl restart nginx
```
> **_NOTE:_**  Comment out the default server block in the `/etc/nginx/sites-enabled/default` file or delete it.

## Setup supervisor

Supervisor will be used to manage the running of CKAN and workers

Navigate to the supervisor configuration folder

```bash
cd /etc/supervisor/conf.d
```

Here you need to add a new [configuration file](../supervisor/ckan-uwsgi.conf) for CKAN and optionaly the [jobworker file](../supervisor/ckan-jobworker.conf) and then update the supervisor

```bash
sudo supervisorctl restart all
```

## Finalizing the setup

Navigate to the CKAN configuration folder

```bash
cd /etc/ckan/default
```

Here you can add all the required configurations to the `ckan.ini` file. After adding the [configurations](../etc-ckan-default/), restart the CKAN server

Now you can access CKAN at `http://<your-server-ip>`
