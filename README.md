# Mesos Continuous Integration and Continuous Deployment Pipeline
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

Weave Install
=============
With the application spread out on the Mesos cluster it is important to have a multi-host overlay network.
This can be achieved with Dockers native multi-host networking or Weave.
Some entry level guide (with much information missing) can be found here [Weave Mesos Marathon Guide](http://weave.works/guides/platform/mesos-marathon/os/centos/cloud/vagrant/index.html).

To install weave run:

```
curl -L git.io/weave -o /usr/local/bin/weave
chmod a+x /usr/local/bin/weave
```

Next the services have to be installed (can be found in this repository). They have to be copied to /etc/systemd/system/ and then following commands have to be run (includes the configuration of the peers that participate in the overlay network):

```
echo 'WEAVE_PEERS="mesos01c7 mesos02c7 mesos03c7"' > /etc/weave.env
systemctl daemon-reload
systemctl enable weave.service weaveproxy.service
systemctl start weave.service weaveproxy.service
```

To use Weave locally (not from Marathon / Mesos) you will have to set the correct environment:

```
eval "$(weave env)"
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
The Docker Registry allows you to host your own images which is used in this case to build and publish the images from jenkins and then also pull them from there and deploy them on Mesos / Marathon.

First you will need a set of certificates:

```
mkdir /certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /certs/registry.key -out /certs/registry.crt
```

Next you have to start the registry container:

```
docker run -d -p 5000:5000 --restart=always --name registry -v `pwd`/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key registry:2
```

And as a last step you will have to trust the certificates from all hosts which also requires a docker service restart. $DTR stands for the hostname of your registry host, in my case this was mesos01c7:

```
openssl s_client -connect $DTR:5000 -showcerts </dev/null 2> /dev/null | openssl x509 -outform PEM | sudo tee /etc/pki/ca-trust/source/anchors/$DTR.crt
update-ca-trust
service docker restart
```

Jenkins Install
===============
For Jenkins it is again easiest to use a Docker container (for the master). There are also Docker containers for the slaves and the option to run "Docker in Docker", but it worked much more stable with running the Slave on one of the Mesos hosts (and is close to no setup work to make this happen).
You could also use REX-RAY already at this step to utilize (and create on first use) a volume on ScaleIO.

```
docker run -d --restart=always -p 8081:8080 -p 50000:50000 -v /home/jenkins:/var/jenkins_home -u root jenkins
```

For Jenkins you need two plugins (these can be installed from the Jenkins UI):

```
1. Cloudbees Docker build and publish plugin
2. Git plugin
```

Adding the App
==============
I took the example from [DockerCon EU Tutorials](https://github.com/docker/dceu_tutorials/blob/master/9-cicd-with-docker.md) and modified it to use Marathon / Mesos and Weave multi-host overlay networks instead of Swarm and Compose with Docker multi-host networking.
The changed code is part of this repository unter voteapps.

Marathon Deploy Tool
====================
It is possible to use curl to send a JSON to Marathon and this can be easily done from Jenkins. Unfortunatelly this has some drawbacks:

1. No (easy) way to check if the deployment really succeeded (unless you would parse the response)
2. No support for environment variables within JSON files

As i needed both i used the Marathon Deploy Ruby Gem from here [GitHub Marathon Deploy](https://github.com/eBayClassifiedsGroup/marathon_deploy) which supports YAML, automatically checks the success of the deployment and also support environment variables.

Installing is simple:

```
yum install gem
gem install marathon_deploy
```

KNOWN ISSUES
============
* REX-RAY service does not always start (fails), manual restart works (most likely wrong start order due to missing dependency definitions, e.g. for ScaleIO SDC)
* Only one overlay network possible, should use Weave as plugin or use native Docker multi-host networking to overcome this limitation
* use REX-RAY in the app for redis and postgres again
* use REX-RAY for jenkins and registry (or even better use ECS via S3 for registry)
* check if docker/example-voting-app might works as well or even be a better choice
