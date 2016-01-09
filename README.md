# MesosCICD
Mesos, Marathon, Jenkins, Docker, Weave, Docker Registry, ScaleIO, RexRay and a little App :)

I started with a Centos 7 install. Make sure to use Centos 7 or newer, as older kernels will not allow you to have Docker 1.9, which is a prerequisite for Volume Driver support and others.
I used Centos 7 Minimal Image which was just fine for what is needed.
Next install some tools that make the networkmgmt. easier.

`<yum install net-tools vim>`

In case you decide to run these as VMs, i can highly recommend to create a template and fix eth0 for your clones (if you do not have eth0 it should all be fine already).
To do so in the template (before shutdown and clone):

1. remove UUID and MAC from vim /etc/sysconfig/network-scripts/ifcfg-eth0
2. rm -f /etc/udev/rules.d/70-persistent-net.rules

Next you should disable the firewall (proper thing here would be to add all the necessary exceptions):

```
systemctl disable firewalld
systemctl stop firewalld
```

Ensure to set the hostname per server (my hosts are named mesos01c7 mesos02c7 mesos03c7):

`<hostnamectl set-hostname mesos01c7>`

Make sure to have DNS entries for all your server (you will need at least 3 server). 
Or modify the hosts file for each one.

`<vi /etc/hosts>`

Docker Install
==============
This comes first, as several services later will be deployed as docker containers.
So let's make sure we have the latest updates:

`<yum update>`

Add the docker repository:

```
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

Install docker engine:

```
yum install docker-engine
```

And start the service, as well as enable it for autostart:

```
service docker start
systemctl enable docker.service
```

Mesos and Marathon Install
=============
A great guide can be found at [Mesosphere Getting Started](https://open.mesosphere.com/getting-started/install/).

In this case I decided to have slave and master on the same host:

```
sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
sudo yum -y install mesos marathon mesosphere-zookeeper
```

Each server needs an unique ID (value between 1 and 255):

```
echo '1' > /var/lib/zookeeper/myid
```

Make sure to extend the configuration for Zookeeper with your servers:

```
sudo tee /etc/zookeeper/conf/zoo.cfg <<-'EOF'
server.1=<mesos_server_1>:2888:3888
server.2=<mesos_server_2>:2888:3888
server.3=<mesos_server_3>:2888:3888
EOF
```

```
echo 'zk://<mesos_server_1>:2181,<mesos_server_2>:2181,<mesos_server_3>:2181/mesos' > /etc/mesos/zk
```

With 3 servers the quorum should be set to 2:

```
echo '2' > //etc/mesos-master/quorum
```

And finaly start the service (or restart if already running):

```
sudo systemctl start zookeeper
```

Next we setup Mesos and Marathon to use Docker and Weave. Most of the content can be found at [Mesosphere Docker Support](https://mesosphere.github.io/marathon/docs/native-docker.html)

For Docker we have to add the containerizer and extend the timeouts to give time for docker image pulls:

```
echo 'docker,mesos' > /etc/mesos-slave/containerizers
echo '5mins' > /etc/mesos-slave/executor_registration_timeout
```

To enable Weave from MEsos / Marathon all that is needed is to change the docker socket to use:

```
echo '/var/run/weave/weave.sock' > /etc/mesos-slave/docker_socket
```

You can open a browser and point it to Port 5050 for Mesos and 8080 for Marathon.
Additionally you can test everything with:

```
MASTER=$(mesos-resolve `cat /etc/mesos/zk`)
mesos-execute --master=$MASTER --name="cluster-test" --command="sleep 5"
```

Or even post a new application to Marathon (example.json is part of this repository):

```
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d@example.json
```

ScaleIO SDC Install
===================
The ScaleIO cluster in this case was outside of the environment, so only the SDC (client) is needed per host to be able to mount volumes into the containers.

ScaleIO needs some additional libraries:
`<yum install numactl>`

Next install the rpm for Centos 7 (i am using latest 1.32.3 release of ScaleIO). Get it for free from [ScaleIO Free & Frictionless](http://emc.com/getScaleIO).

`<rpm -Uvh /tmp/EMC-ScaleIO-sdc-1.32-3455.5.el7.x86_64.rpm>`

If you used templates and had the SDC already included it is important to reset the IDs (not needed for per node install):

`<rm -f /bin/emc/scaleio/drv_cfg.txt>`
`<rm -f /etc/emc/scaleio/ini_guid>`

And now you'll have to join it with the ScaleIO cluster (IPs provided are for the two MDMs)

`</opt/emc/scaleio/sdc/bin/drv_cfg --add_mdm --ip <mdm_ip_1>,<mdm_ip_2>>`

REX-RAY Install
===============
Next we install REX-RAY, the volume driver for Docker that integrates native with ScaleIO.

```
curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -
```

An example configuration file (/etc/rexray/config.yml) is part of this repository.
Next start and enable the rexray service

```
rexray service start
systemctl enable rexray.service
```

DOCKER REGISTRY INSTALL
=======================
The Docker Registry allows
install registry on centos 7:

mkdir /certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /certs/registry.key -out /certs/registry.crt
docker run -d -p 5000:5000 --restart=always --name registry -v `pwd`/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key registry:2

on all nodes:
openssl s_client -connect $DTR:5000 -showcerts </dev/null 2> /dev/null | openssl x509 -outform PEM | sudo tee /etc/pki/ca-trust/source/anchors/$DTR.crt
update-ca-trust
service docker restart
----------
enable docker multihost networking

install a keyvalue store for global configs
docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap

/usr/lib/systemd/system/docker.service add daemon flags
--cluster-store=consul://mesos01c7:8500/network --cluster-advertise=ens160:2375

systemctl daemon-reload
systemctl restart docker.service

ISSUE: never worked !!
------------
setting up jenkins and builds with docker

docker run -d --restart=always -p 8081:8080 -p 50000:50000 -v /home/jenkins:/var/jenkins_home -u root jenkins

install "cloudbees docker build and publish plugin" and "git plugin"
need a mesos slave cause can't run docker inside docker (master jenkins is a docker container) for docker build

config of build job:


-----------
install git for dev example and push to jenkins with auto build of container !
examples:
https://clusterhq.com/2015/08/28/flying-with-marathon/
http://container-solutions.com/continuous-delivery-with-docker-on-mesos-in-less-than-a-minute-part-2/
https://github.com/docker/dceu_tutorials/blob/master/9-cicd-with-docker.md


to get rid of application group:
curl -X DELETE -H "Content-type: application/json" localhost:8080/v2/groups/voteapps
-----------
and we need overlay networks as well, so we need weave:
http://weave.works/guides/platform/mesos-marathon/os/centos/cloud/vagrant/index.html

curl -L git.io/weave -o /usr/local/bin/weave
chmod a+x /usr/local/bin/weave

on primary node:
weave launch

on other nodes:
weave launch mesos01c7

echo 'WEAVE_PEERS="mesos01c7 mesos02c7 mesos03c7"' > /etc/weave.env
add weave.service and weaveproxy.service to /etc/systemd/system/
enable and start them

use weave locally (not from mesos):
eval "$(weave env)"

expose weave isolated containers (not needed as we have them in the weave and bridge network of docker at the same time => combine with port forwards):
weave expose (gives the mesos hosts an ip address)
iptables -t nat -A PREROUTING -p tcp -i ens160 --dport 50080 -j DNAT --to-destination $(weave dns-lookup voting-app):80
iptables -t nat -A PREROUTING -p tcp -i ens160 --dport 50081 -j DNAT --to-destination $(weave dns-lookup result-app):80
--------
to expose env variables for deployments to marathon we need yaml support (ruby gem available)
https://github.com/eBayClassifiedsGroup/marathon_deploy

yum install gem
gem install marathon_deploy

=> super cool as it also checks on progress / success of deployment !!

-------
docker connect/attach: docker exec -i -t 665b4a1e17b6 bash

-------
SIDE NOTE: grow FS
- grow root disk in VMWare
- fdisk /dev/sda
-- remove partition, create partition (will have extended size) -> sda2
- pvresize /dev/sda2
- lvextend -l +100%FREE /dev/centos/root
- xfs_growfs -d /dev/centos/root

----------
KNOWN ISSUES:
- host with sio gateway is critical; hosts also have SIO on them, so can't restart two of them!!
- rexray service does not start (fails), manual restart works (goes have to load SDC first or such)
- 
