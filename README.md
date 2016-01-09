# MesosCICD
Mesos, Marathon, Jenkins, Docker, Weave, Docker Registry, ScaleIO, RexRay and a little App :)

I started with a Centos 7 install. Make sure to use Centos 7 or newer, as older kernels will not allow you to have Docker 1.9, which is a prerequisite for Volume Driver support and others.
I used Centos 7 Minimal Image which was just fine for what is needed.
Next install some tools that make the networkmgmt. easier.
<yum install net-tools vim>

rm -f /etc/udev/rules.d/70-persistent-net.rules
remove UUID and MAC, etc.
vim /etc/sysconfig/network-scripts/ifcfg-ens160
------
systemctl disable firewalld
systemctl stop firewalld

needs DNS entries => added to hosts file
fix hostnames:
hostnamectl set-hostname mesos01c7
vi /etc/hosts

mesos install:
https://open.mesosphere.com/getting-started/install/
sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
sudo yum -y install mesos marathon mesosphere-zookeeper

------
SDC install
yum install numactl
rm -f /bin/emc/scaleio/drv_cfg.txt
rm -f /etc/emc/scaleio/ini_guid
rpm -e EMC-ScaleIO-sdc-1.32-3455.5.el7.x86_64
rpm -Uvh /tmp/EMC-ScaleIO-sdc-1.32-3455.5.el7.x86_64.rpm
/opt/emc/scaleio/sdc/bin/drv_cfg --add_mdm --ip 192.168.2.41,192.168.2.42
---------------
docker install:
yum update

sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

yum install docker-engine

service docker start
systemctl enable docker.service

----------
install and configure REX-Ray:
curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -
using /etc/rexray/config.yml
rexray service start
---------
setup mesos + marathon for docker:
https://mesosphere.github.io/marathon/docs/native-docker.html
echo 'docker,mesos' > /etc/mesos-slave/containerizers
if you want to use weave, also add
echo '/var/run/weave/weave.sock' > /etc/mesos-slave/docker_socket
Increase the executor timeout to account for the potential delay in pulling a docker image to the slave.
echo '5mins' > /etc/mesos-slave/executor_registration_timeout

--------

/etc/zookeeper/conf/zoo.cfg
append: 
server.1=1.1.1.1:2888:3888
server.2=2.2.2.2:2888:3888
server.3=3.3.3.3:2888:3888

/etc/mesos/zk
set:
zk://192.168.2.81:2181,192.168.2.82:2181,192.168.2.83:2181/mesos

/etc/mesos-master/quorum
set: 
2

unique per node:
set /var/lib/zookeeper/myid (value between 1 and 255)


start services:
restart
sudo systemctl start zookeeper
-------------

http://<master-ip>:5050 
http://<master-ip>:8080

MASTER=$(mesos-resolve `cat /etc/mesos/zk`)
mesos-execute --master=$MASTER --name="cluster-test" --command="sleep 5"

----------
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d@/mesosconfigs/redis.json

-------
start rexray service
systemctl start rexray.service
----------------------------
install DTR on Centos 7: => FAILS FAILS FAILS !!
docker run docker/trusted-registry install | \bash

needed on all hosts using the DTR:
export DTR=mesos01c7
openssl s_client -connect $DTR:443 -showcerts </dev/null 2> /dev/null | openssl x509 -outform PEM | sudo tee /etc/pki/ca-trust/source/anchors/$DTR.crt
update-ca-trust
service docker restart
docker login https://$DTR
-------------
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
