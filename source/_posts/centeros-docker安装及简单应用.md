---
title: centeros docker安装及简单应用
tags:
  - docker
  - jenkins
categories:
  - docker
  - CI/CD
date: 2019-08-16 12:52:02
img: /gallery/thumbnails/docker.png
summary: docker安装及简单应用
---

#### docker安裝
<!--more-->
##### 卸载
```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```
##### 安裝
默认安装stable版的，安装edge或者test版本的请自行查阅[官方文档](https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce-1)
```
#安装配置管理工具
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
 sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
 sudo yum install -y docker-ce
```
##### 安装指定版本

```
yum install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.0.ce-1.el7.centos.noarch.rpm
yum install -y docker-ce-17.03.0.ce-1.el7.centos.x86_64
```

##### docker 加速配置（centos7)
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://id7d29lp.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.1.38:5000"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
###### 配置非root用户使用docker

```
#bamboo 替换成你的用户名
sudo setfacl -m user:bamboo:rw /var/run/docker.sock
#还有一种方式就是在加入docker的group

```
##### login
```bash
docker login
#你的docker hub的用户名密码
```
用户名为参数登录
```bash
docker login --username=gedit registry.cn-hangzhou.aliyuncs.com
# 你的ali dockerhub的密码
```
#### docker machine

```
docker-machine create -d generic --generic-ip-address=192.168.1.67 --generic-ssh-user=administrator host67
```
#### swarm集群
##### 配置
```
sudo tee /etc/sysconfig/docker <<-'EOF'
OPTIONS='-g /cutome-path/docker -H tcp://0.0.0.0:2375'
EOF
```
##### 安装swarm

```
docker pull swarm
```

##### 生成token

```
docker -H 192.168.1.38:8888 ps -a
```

##### 设置管理节点
在节点里面管理设置
```
 docker swarm init --advertise-addr 192.168.1.38
```
##### 加入集群
```
docker swarm join \
    --token SWMTKN-1-2hjlzufhptigxwvclhbn2b96rvn2ndl8wy8dqzn8otvjcibydp-7hppr5l9agyyp41wght5gpeo6 \
    192.168.1.41:2377
```

##### 查看集群节点

```
docker node ls
```
##### 执行命令

```
docker service create --name guoi-micro-shopie-shop -p 9039:9002 -p 8940:8902 --env JAVA_OPTS="-Xmx512m" guoi/guoi-micro-shopie-shop --constraint 'node.hostname==istio-master'

```



#### 配置本地仓库
[原文链接](https://blog.csdn.net/ronnyjiang/article/details/71189392)

```
docker run -d -p 5000:5000 -v /home/docker_registry:/var/lib/registry --restart=always --name registry registry:latest  
```

```
# insecure-registries设置本地仓库位置
cat /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://7xwv2psl.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.1.250:5000"]
}
```
查看仓库位置

```
curl -XGET http://192.168.1.250:5000/v2/_catalog
curl -XGET http://192.168.1.250:5000/v2/guoi/guoi-micro-shopie-catalog/tags/list
```


#### jenkins jar运行
```
cd github/gedit_cloud_user_test/
pid=`ps -ef | grep gedit-cloud-user-0.0.1-SNAPSHOT.jar | grep -v grep | awk '{print $2}'`
if [ -n "$pid" ]
then
#!kill -9 强制终止
   echo "kill -9 的pid:" $pid
   kill -9 $pid
fi
ls
gradle build -x test -x docker
BUILD_ID=DONTKILLME #jenkins环境变量
nohup java -Dspring.profiles.active=test -jar -Xmx500m build/libs/gedit-cloud-user-0.0.1-SNAPSHOT.jar >>/root/log/gedit_user/user.log 2>&1&
```
#### docker mysql

```
#dump data
mysql -uroot gedit_store>sqlbackup/gedit_store.docker.sql
mysql -uroot gedit_user>sqlbackup/gedit_user.docker.sql
#close mysql
systemctl mysqld stop
#install docker mysql
docker pull mysql 
#run instance
docker run -itd -p 3306:3306 mysql bash
#login tty
container=$(docker ps|grep mysql|awk '{print $1}')
docker exec -it $container bash
#create databases
mysql -uroot
create database gedit_store character set utf8mb4;
create database gedit_user character set utf8mb4;
exit
#exit tty
exit
#get mysql ip
mysqlIp=$(docker ps|grep mysql|awk '{print $1}'|xargs docker inspect| grep IPAddress|sed -n '2p'|awk '{print $2}'|sed -e 's/\"//g'|sed -e 's/\,//g')
echo "mysql ip:$mysqlIp"
#backup data,need dump mysql data
mysql -uroot -h$mysqlIp gedit_store<sqlbackup/gedit_store.docker.sql
mysql -uroot -h$mysqlIp gedit_user<sqlbackup/gedit_user.docker.sql
```
##### 删除none镜像

```
#delete none images
docker rmi -f $(docker images | grep '^<none>' | awk '{print $3}')
```

##### jenkins docker

```
BUILD_ID=DONTKILLME
gradle build -x test --refresh-dependencies
sed 's/-Dspring.profiles.active=test/-Dspring.profiles.active=test/g' Dockerfile
containerId=$(docker ps -a|grep 'conanchen/gedit-cloud-storesearch'|awk '{print $1}')
if [ $containerId ]
then
	echo "stop container id $containerId"
    upContainerId=$(docker ps -a|grep 'conanchen/gedit-cloud-storesearch'|grep 'Up'|awk '{print $1}')
    if [ $upContainerId ]
    then
    	 docker kill $upContainerId
    fi
    docker rm -f $containerId
    docker rmi -f $(docker images|grep 'conanchen/gedit-cloud-storesearch'|awk '{print $3}')
fi
gradle build docker -x test -x dockerPush
docker run --name gedit_storesearch -d -p 9091:9090 -p 9985:9985 --env JAVA_OPTS="-Xmx512m" conanchen/gedit-cloud-storesearch
sleep 30
docker ps -a|grep 'conanchen/gedit-cloud-storesearch'|awk '{print $1}'|xargs docker logs
```
##### elasticsearch

```
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.1.1
docker run -d -it -p 19200:9200 -p 19300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms256m -Xmx512m" docker.elastic.co/elasticsearch/elasticsearch:6.1.1
```

