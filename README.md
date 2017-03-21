# web-application cluster boiler-plate
Documentation and scripts for running a full web application in a micro-services style. Many pieces are optional and could be swapped out to match others desires.

## Transitioning to Ansible
Currentlly transitioning from fleet to ansible

## Unit Files, Scripts, Playbooks
[![Build Status](https://travis-ci.org/chad-autry/wac-bp.svg?branch=master)](https://travis-ci.org/chad-autry/wac-bp)

The unit files, scripts, and playbooks in the dist directory have been extracted from this document and pushed back to the repo.

## Assumptions and Opinions
* Alpine Linux is my prefered containerized OS, and the choice I've made for images
* CoreOS is the chosen host operating system
* Ansible is used for orchestration

## Requirements and Features
* Dockerized nginx container to host static site
  * Forward's nginx logs to the docker service, [loggly logging strategy article](https://www.loggly.com/blog/top-5-docker-logging-methods-to-fit-your-container-deployment-strategy/)
  * Automatically reconfigures and refreshes nginx config based on routing configuration provided through etcd
  * SSL termination
  * By default forward http connections to https
  * Have a configuration mode which allows initial letsencrypt validation over http
* Https certificate from letsencrypt with autorenewal
* Static Front End boilerplate, with static component upgrade strategy
* Containerized Node.js server with application upgrade strategy
  * Boilerplate to publish to etcd
  * Discovers database from etcd
  * Oauth & Oauth2 Termination
    * JWT generation and validation
* Dockerized  RethinkDB
  * Boilerplate to publish to etcd

## Externalities
* Configure DNS
* Create tagged machine instances
* Create ansible inventory
* Firewall
* Machine instance monitoring

## Fleet Deployment 
### Basic Cloud Config
Just an example. Starts fleet, bootstraps a single static etcd cluster with only the single instance as both frontend and backend
The way I finally loaded it was using the command
sudo coreos-cloudinit --from-file=/home/chad_autry/cloud-config.yaml

```
#cloud-config

coreos:
  etcd2:
    name: etcdserver
    initial-cluster: etcdserver=http://10.142.0.2:2380
    initial-advertise-peer-urls: http://10.142.0.2:2380
    advertise-client-urls: http://10.142.0.2:2379
    listen-client-urls: http://0.0.0.0:2379
    listen-peer-urls: http://0.0.0.0:2380
  fleet:
      public-ip: 10.142.0.2
      metadata: "frontend=true,backend=true"
  units:
    - name: etcd2.service
      command: start
    - name: fleet.service
      command: start
```

### Set Values
Various units expect values to be configured in etcd
```
/usr/bin/etcdctl set /domain/name <domain>
/usr/bin/etcdctl set /domain/email <email>
/usr/bin/etcdctl set /rethinkdb/pwd <Web Authorization Password>
/usr/bin/etcdctl set /node/config/token_secret <Created Private Key>
/usr/bin/etcdctl set /node/config/auth/google/client_id <Google Client ID>
/usr/bin/etcdctl set /node/config/auth/google/redirect_uri <Google Redirect URI>
/usr/bin/etcdctl set /node/config/auth/google/secret <Google OAuth Secret>
```

### Scripts and Files
Download or checkout the project. Within the dist directory there will be helper scripts and the units

This first script submits all the unit in the units directory. Then it starts the ones under started subdirectory.

[start-units.sh](dist/start-units.sh)
```bash
#!/bin/bash

find ./units -type f -exec fleetctl load {} \;
find ./units/started -type f -exec fleetctl start {} \;
```

This second script will destroy all the units, so they can be redeployed

[destroy-units.sh](dist/destroy-units.sh)
```bash
#!/bin/bash

find ./units -type f -exec fleetctl destroy {} \;
```

## Ansible Deployment
The machine used for a controller will need SSH access to all the machines being managed. You can use one of the instances being managed, on GCE [cloud shell](https://cloud.google.com/shell/docs/) is a handy resource to use.

If docker is available, [containerized Ansible](https://github.com/chad-autry/wac-ansible) can be used, else you'll need to install Python and Ansible onto the controller.

```
docker run -it --net host -v$(pwd):/ansible/playbooks chadautry/wac-ansible -i <inventory file> <playbook>
```

### Ansible Inventory
Here is an example inventory. wac-bp operates on machines based on the group they belong to. You can manually create the inventory file with the hosts to manage.

```
hostnameone
hostnametwo

[etcd]
hostnameone

[rethinkdb]
hostnameone

```

#### Dynamic Inventory
Even better tag the instances at creation, and use a dynamic inventory script. From your cloud shell instance

```
cd ~/ansible/inventory
wget --no-check-certificate https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/gce.ini
wget --no-check-certificate https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/gce.py
```

Then edit the gce_project_id = field for your project. We'll leave the rest blank since we're on a GCE instance and it will automatically use the service account.

### group_vars/all
The variables file contains all the container versions to use.

[group_vars/all](dist/ansible/group_vars/all)
```yaml
wac-python.version=latest
```

### Python
Ansible requires Python on the remote instances to run the majority of its playbook commands. The included python playbook uses raw commands and can be used to setup a containerized Python on all the instances.

```
docker run -it --net host -v $(pwd):/var/ansible chadautry/wac-ansible -i ./inventory ./playbooks/setupPython.yaml
```
[setupPython.yaml](dist/ansible/playbooks/setupPython.yaml)
```yaml
---
- hosts: all:!localhost
  remote_user: root
  tasks:
  - name: Ensure /opt/bin exists
    raw: mkdir -p /opt/bin
  - name: Pull container
    raw: docker pull chadautry/wac-python:{wac-python.version}
  - name: Instantiate container
    raw: docker create --name copy-python chadautry/wac-python
  - name: Copy script from container instance
    raw: docker cp copy-python:/opt/bin/python.sh /opt/bin
  - name: Delete container instance
    raw: docker rm copy-python
  - name: Set permissions on script
    raw: chmod 755 /opt/bin/python.sh
```

## Frontend Units
### nginx unit
The main unit for the front end, nginx is the static file server and reverse proxy. Can have redundant identical instances.

[nginx.service](dist/units/started/nginx.service)
```yaml
[Unit]
Description=NGINX
# Dependencies
Requires=docker.service

# Ordering
After=docker.service

[Service]
ExecStartPre=-/usr/bin/docker pull chadautry/wac-nginx
ExecStartPre=-/usr/bin/docker rm nginx
ExecStart=/usr/bin/docker run --name nginx -p 80:80 -p 443:443 \
-v /var/www:/usr/share/nginx/html:ro -v /var/ssl:/etc/nginx/ssl:ro \
-v /var/nginx:/usr/var/nginx:ro \
chadautry/wac-nginx
Restart=always

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* requires docker
* wants all files to be copied before startup
* Starts a customized nginx docker container
    * Takes server config from local drive
    * Takes html from local drive
    * Takes certs from local drive
* Blindly runs on all frontend tagged instances

### nginx reloading units
A pair of units are responsible for reloading nginx instances on file changes

[nginx-reload.service](dist/units/nginx-reload.service)
```yaml
[Unit]
Description=NGINX reload service

[Service]
ExecStart=-/usr/bin/docker kill -s HUP nginx
Type=oneshot

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* Sends a signal to the named nginx container to reload
* Ignores errors
* It is a one shot which expects to be called by other units
* Metadata will cause it to be made available on all frontend servers when loaded

[nginx-reload.path](dist/units/started/nginx-reload.path)
```yaml
[Unit]
Description=NGINX reload path

[Path]
PathChanged=/var/nginx/nginx.conf
PathChanged=/var/ssl/fullchain.pem

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* Watches config file
* Watches the (last copied) SSL cert file
* Automatically calls nginx-reload.service on change (because of matching unit name)
* Blindly runs on all frontend tagged instances

### SSL
With nginx in place, several units are responsible for updating its SSL certificates

#### acme challenge response watcher
This unit takes the acme challenge response from etcd, and templates it into the nginx config

[acme-response-watcher.service](dist/units/started/acme-response-watcher.service)
```yaml
[Unit]
Description=Watches for distributed acme challenge responses
# Dependencies
Requires=etcd2.service

# Ordering
After=etcd2.service

[Service]
ExecStartPre=-/usr/bin/docker pull chadautry/wac-nginx-config-templater
ExecStartPre=-/usr/bin/docker rm nginx-templater
ExecStart=/usr/bin/etcdctl watch /acme/watched
ExecStartPost=-/usr/bin/docker run --name nginx-templater --net host \
-v /var/nginx:/usr/var/nginx -v /var/ssl:/etc/nginx/ssl:ro \
chadautry/wac-nginx-config-templater
Restart=always

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* Starts a watch for changes in the acme challenge response
* Once the watch starts, executes the config templating container
    * Local volume mapped in for the templated config to be written to
    * Doesn't error out (TODO move to a secondary unit to be safely concurrent)
* If the watch is ever satisfied, the unit will exit
* Automatically restarted, causing a new watch and templater execution
* Blindly runs on all frontend tagged instances

#### SSL Certificate Syncronization
This unit takes the SSL certificates from etcd, and writes them to the local system

[certificate-sync.service](dist/units/started/certificate-sync.service)
```yaml
[Unit]
Description=SSL Certificate Syncronization
# Dependencies
Requires=etcd2.service

# Ordering
After=etcd2.service

[Service]
ExecStartPre=-mkdir /var/ssl
ExecStart=/usr/bin/etcdctl watch /ssl/watched
ExecStartPost=/bin/sh -c '/usr/bin/etcdctl get /ssl/server_chain > /var/ssl/chain.pem'
ExecStartPost=/bin/sh -c '/usr/bin/etcdctl get /ssl/key > /var/ssl/privkey.pem'
ExecStartPost=/bin/sh -c '/usr/bin/etcdctl get /ssl/server_pem > /var/ssl/fullchain.pem'
Restart=always

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* Starts a watch for SSL cert changes to copy
* Once the watch starts, copy the current certs
* If the watch is ever satisfied, the unit will exit
* Automatically restarted, causing a new watch and copy
* Metadata driven, don't bother with binding

#### letsencrypt renewal units
A pair of units are responsible for initiating the letsencrypt renewal process each month

[letsencrypt-renewal.service](dist/units/letsencrypt-renewal.service)
```yaml
[Unit]
Description=Letsencrpyt renewal service

[Service]
ExecStartPre=-/usr/bin/docker pull chadautry/wac-acme
ExecStartPre=-/usr/bin/docker rm acme
ExecStart=-/usr/bin/docker run --net host --name acme chadautry/wac-acme
Type=oneshot

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* Calls the wac-acme container to create/renew the cert
    * Container takes domain name and domain admin e-mail from etcd
    * Container interacts with other units/containers to manage the renewal process through etcd
* Ignores errors
* It is a one shot which expects to be called by the timer unit
* Metadata will cause it to be made available on all frontend servers when loaded
    * It technically could run anywhere with etcd, just limiting its loaded footprint

> Special Deployment Note:
> * Put the admin e-mail and domain into etcd
> ```
> /usr/bin/etcdctl set /domain/name <domain>
> /usr/bin/etcdctl set /domain/email <email>
> ```
> * Manually run the wac-acme container once to obtain certificates the first time
> ```
> sudo docker run --net host --name acme chadautry/wac-acme
> ```

[letsencrypt-renewal.timer](dist/units/started/letsencrypt-renewal.timer)
```yaml
[Unit]
Description=Letsencrpyt renewal timer

[Timer]
OnCalendar=*-*-01 05:00:00
RandomizedDelaySec=60

[X-Fleet]
MachineMetadata=frontend=true
```
* Executes once a month on the 1st at 5:00
    * Avoid any DST confusions by avoiding the midnight hours
* Assuming this gets popular (yeah right), add a 1 minute randomized delay to not pound letsencrypt
* Automagically executes the letsencrypt-renewal.service based on name
* Not global so there will only be one instance

### backend discovery unit
Sets a watch on the backend discovery location, and when it changes templates out the nginx conf

[backend-discovery-watcher.service](dist/units/started/backend-discovery-watcher.service)
```yaml
[Unit]
Description=Watches for backened instances
# Dependencies
Requires=etcd2.service

# Ordering
After=etcd2.service

[Service]
ExecStartPre=-/usr/bin/docker pull chadautry/wac-nginx-config-templater
ExecStartPre=-/usr/bin/docker rm nginx-templater
ExecStart=/usr/bin/etcdctl watch /discovery/backend
ExecStartPost=-/usr/bin/docker run --name nginx-templater --net host \
-v /var/nginx:/usr/var/nginx -v /var/ssl:/etc/nginx/ssl:ro \
chadautry/wac-nginx-config-templater
Restart=always

[X-Fleet]
Global=true
MachineMetadata=frontend=true
```
* Starts a watch for changes in the backend discovery path
* Once the watch starts, executes the config templating container
    * Local volume mapped in for the templated config to be written to
    * Doesn't error out (TODO move to a secondary unit to be safely concurrent)
* If the watch is ever satisfied, the unit will exit
* Automatically restarted, causing a new watch and templater execution
* Blindly runs on all frontend tagged instances

## API Backend
These are the units for an api backend, including authentication. A cluster could have multiple backend processes, just change the tagging from 'backend' to some named process (and change the docker process name)

#### node config watcher
This unit watches the node config values in etcd, and templates them to a file for the node app when they change

[node-config-watcher.service](dist/units/started/node-config-watcher.service)
```yaml
[Unit]
Description=Watches for node config changes
# Dependencies
Requires=etcd2.service

# Ordering
After=etcd2.service

[Service]
ExecStartPre=-/usr/bin/docker pull chadautry/wac-node-config-templater
ExecStartPre=-/usr/bin/docker rm node-templater
ExecStart=/usr/bin/etcdctl watch --recursive /node/config
ExecStartPost=-/usr/bin/docker run --name node-templater --net host \
-v /var/nodejs:/usr/var/nodejs -v /var/ssl:/etc/nginx/ssl:ro \
chadautry/wac-node-config-templater
ExecStartPost=-/usr/bin/docker stop backend-node-container
Restart=always

[X-Fleet]
Global=true
MachineMetadata=backend=true
```
* Starts a watch for changes in the top node config path
* Once the watch starts, executes the config templating container
    * Local volume mapped in for the templated config to be written to
    * Doesn't error out
* Once the config is templated, stops the node container (it will be restarted by its unit)
* If the watch is ever satisfied, the unit will exit
* Automatically restarted, causing a new watch and templater execution
* Blindly runs on all backend tagged instances

> Special Deployment Note:
> * Put the node authorization config values into etcd
> ```
> /usr/bin/etcdctl set /node/config/token_secret <Created Private Key>
> /usr/bin/etcdctl set /node/config/auth/google/client_id <Google Client ID>
> /usr/bin/etcdctl set /node/config/auth/google/redirect_uri <Google Redirect URI>
> /usr/bin/etcdctl set /node/config/auth/google/secret <Google OAuth Secret>
> ```

### nodejs unit
The main application unit, it is simply a docker container with Node.js installed and the code to be executed mounted inside

[nodejs.service](dist/units/started/nodejs.service)
```yaml
[Unit]
Description=NodeJS Backend API
# Dependencies
Requires=docker.service
Requires=node-config-watcher.service

# Ordering
After=docker.service
After=node-config-watcher.service

[Service]
ExecStartPre=-/usr/bin/docker pull chadautry/wac-node
ExecStartPre=-/usr/bin/docker rm -f backend-node-container
ExecStart=/usr/bin/docker run --name backend-node-container -p 8080:80 -p 4443:443 \
-v /var/nodejs:/app:ro \
chadautry/wac-node
ExecStop=-/usr/bin/docker stop backend-node-container
Restart=always

[X-Fleet]
Global=true
MachineMetadata=backend=true
```
* requires docker
* requires the configuration templater
* Starts a customized nodejs docker container
    * Takes the app from local drive
* Blindly runs on all backend tagged instances

### nodejs code update unit
* Need to distribute code accross all backend instances
* Need to restart the local Node server when new code is on the machine

### backend publishing unit
Publishes the backend host into etcd at an expected path for the frontend to route to

[backend-publishing.service](dist/units/started/backend-publishing.service)
```yaml
[Unit]
Description=Backend Publishing
# Dependencies
Requires=etcd2.service

# Ordering
After=etcd2.service

[Service]
ExecStart=/bin/sh -c "while true; do etcdctl set /discovery/backend/%H '%H' --ttl 60;sleep 45;done"
ExecStop=/usr/bin/etcdctl rm /discovery/backend/%H

[X-Fleet]
Global=true
MachineMetadata=backend=true
```
* requires etcd
* Publishes host into etcd every 45 seconds with a 60 second duration
* Deletes host from etcd on stop
* Blindly runs on all backend tagged instances

### rethinkdb proxy unit
A rethinkdb proxy on localhost for nodejs units to connect to

[rethinkdb-proxy.service](dist/units/started/rethinkdb-proxy.service)
```yaml
[Unit]
Description=RethinkDB Proxy
# Dependencies
Requires=docker.service
Requires=etcd2.service

# Ordering
After=docker.service
After=etcd2.service

[Service]
ExecStartPre=-mkdir /var/rethinkdbproxy
ExecStartPre=-/usr/bin/docker pull chadautry/wac-rethinkdb-config-templater
ExecStartPre=-/usr/bin/docker run \
--net host --rm \
-v /var/rethinkdbproxy:/usr/var/rethinkdb \
chadautry/wac-rethinkdb-config-templater "emptyHost"
ExecStartPre=-/usr/bin/docker pull chadautry/wac-rethinkdb
ExecStartPre=-/usr/bin/docker rm -f rethinkdb-proxy
ExecStartPre=-rm /var/rethinkdbproxy/pid_file
ExecStart=/usr/bin/docker run --name rethinkdb-proxy \
-v /var/rethinkdbproxy:/usr/var/rethinkdb \
-p 29017:29015 -p29018:29016 -p 8082:8080 \
chadautry/wac-rethinkdb proxy --config-file /usr/var/rethinkdb/rethinkdb.conf
Restart=always

[X-Fleet]
Global=true
MachineMetadata=backend=true
```
* requires docker
* Configures the instance, will not exclude any host from join list
* Pulls the image
* Removes the container
* Starts a rethinkdb container in proxy mode
* All ports shifted by 2, so it won't conflict with a non-proxy node
* Blindly runs on all backend tagged instances

## Database

### rethinkdb unit
A rethinkdb node

[rethinkdb.service](dist/units/started/rethinkdb.service)
```yaml
[Unit]
Description=RethinkDB
# Dependencies
Requires=docker.service
Requires=etcd2.service

# Ordering
After=docker.service
After=etcd2.service

[Service]
ExecStartPre=-mkdir /var/rethinkdb
ExecStartPre=-/usr/bin/docker pull chadautry/wac-rethinkdb-config-templater
ExecStartPre=-/usr/bin/docker run \
--net host --rm \
-v /var/rethinkdb:/usr/var/rethinkdb \
chadautry/wac-rethinkdb-config-templater %H
ExecStartPre=-/usr/bin/docker pull chadautry/wac-rethinkdb
ExecStartPre=-/usr/bin/docker rm -f rethinkdb
ExecStartPre=-rm /var/rethinkdb/pid_file
ExecStart=/usr/bin/docker run --name rethinkdb \
-v /var/rethinkdb:/usr/var/rethinkdb \
-p 29015:29015 -p29016:29016 -p 8081:8080 \
chadautry/wac-rethinkdb --config-file /usr/var/rethinkdb/rethinkdb.conf
Restart=always

[X-Fleet]
Global=true
MachineMetadata=database=true
```
* requires docker
* Configures the instance, will exclude its own host from the join list
* Pulls the image
* Removes the container
* Starts a rethinkdb container
* HTTP shifted to 8081 so it won't conflict with nginx if colocated
* Blindly runs on all database tagged instances

> DB Instance Prep:
> There is some prep that needs to be done manually on each DB instance
> ```
> docker run --rm -v /var/rethinkdb:/usr/var/rethinkdb chadautry/wac-rethinkdb create -d /var/rethinkdb/data
> sudo etcd set /discovery/database/<host> <host>
> ```
