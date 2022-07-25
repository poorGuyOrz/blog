---
title: "Envs"
date: 2022-02-21T19:00:25+08:00
draft: false
---

## hugo 

wget https://github.com/gohugoio/hugo/releases/download/v0.92.1/hugo_0.92.1_Linux-64bit.deb
https://github.91chi.fun/https://github.com//gohugoio/hugo/releases/download/v0.92.2/hugo_extended_0.92.2_Linux-64bit.deb
sudo dpkg -i hugo*.deb


140.82.113.3

aria2c -s 5 https://github.com/gohugoio/hugo/releases/download/v0.92.1/hugo_0.92.1_Linux-64bit.deb

https://github.com/gohugoio/hugo/releases/download/v0.92.1/hugo_0.92.1_Linux-64bit.deb

可以使用aria2下载，ubuntu使用`apt install aria2`直接安装工具，使用-s开启多路下载
aria2c -s 5 https://github.com/gohugoio/hugo/releases/download/v0.92.1/hugo_0.92.1_Linux-64bit.deb


manager用户

https://www.jianshu.com/p/a76a93e8c662


## Unix

分区问题，集群上多块磁盘分区挂载到指定目录

```
fdisk disk 可以对一个磁盘进行分区的添加和删除等操作
p 
d
w
h
```

添加磁盘挂载

```
lsblk -f 查看磁盘
mkfs.xfs -f -n ftype=1 /dev/sdb1 格式化磁盘
mount /dev/sdb1 /var/lib/docker  挂载
xfs_info /dev/sdb1 | grep ftype=1
blkid /dev/sdb1                  查看UUID
UUID=<UUID> /var/lib/docker xfs defaults 0 0 写进/etc/fstab
```

问题`Couldn't find device with uuid 4mhUbb-Ls1h-jp0d-JuJK-C38V-T3tX-f7s2IN`

原因未知，疑似但是这个UUID是前面挂载的分区格式化之前的UUID，所以可能是挂载的时候出了什么问题，但是之前对其他机器操作无问题，此问题只出现在集群中的一两太机器上。

解决：

```
使用 vgreduce --removemissing --verbose lvname 解决，但是需要视情况而定，需要知道自己在做什么
其中会使用的命令
	lsblk
	vgscan
	pvscan
	cat /etc/lvm/archive/* | less   查看UUID和盘符之前的关系
	
lvextend -L +300G /dev/mapper/centos-root  扩展分区大小
xfs_growfs /dev/mapper/centos-root         扩展且生效
lvremove /dev/mapper/centos-home           删除lv
lvcreate -L 100G -n /dev/mapper/centos-home	创建lv
mkdf.sfx /xxxx								创建文件系统
```

详细储备知识 [linux 文件](https://blog.csdn.net/lemontree1945/article/details/79293390)



## Ubuntu安装typora

上海无法直连外网，所以这些都需要翻墙，或者使用其他的链接方式，

1. 直接使用snap安装，一般情况下，Ubuntu上可以使用apt安装就使用apt安装，这样节约空间，但是无法使用apt的时候，可以尝试使用snap，不是首选，但是无法链接网络的时候，这是首选方案
2. 使用安装包，但是可能无法下载
3. 添加typora的apt源，使用apt下载，可能有网络问题





## kubelet 换包

[官网](http://docs.kubernetes.org.cn/227.html)

### 角色

* deployment
  * 
* replicaset
* pod




```
docker pull 172.16.1.99/postcommit/inceptor@sha256:d9a8c9c1cf4e1587e2ac468942a987ee43b92c99787f9d392b81a8e1bda26932
docker tag 3b2a1923a5d3 baisheng5:5000/transwarp/argodb-inceptor:argodb-3.2.0-final-wl
docker push baisheng5:5000/transwarp/argodb-inceptor:argodb-3.2.0-final-wl

vdevelop-shiva1-202112141040

kubectl get deployment | grep shiva

172.16.1.99/postcommit/shiva-tabletserver:vdevelop-shiva1-202112141040
172.16.1.99/postcommit/shiva-master:vdevelop-shiva1-202112141040
172.16.1.99/postcommit/shiva-webserver:vdevelop-shiva1-202112141040


```


改变deployment的配置文件的镜像，他会自动重启然后拉取镜像启动服务，所以需要准备自己的镜像，一般只需要替换镜像中的一个或者几个文件，然后让k8s使用此镜像即可，但是无法直接替换镜像中的文件，需要在运行的容器中替换，然后再以此为基准建立新的镜像，然后替换


安装最新gcc g++
https://www.jianshu.com/p/
https://cloud.tencent.com/developer/article/1635218

## Ubuntu换源
```
cd /etc/apt/
sudo cp sources.list sources.list.bak
sudo vim sources.list

注意换源的版本

deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse


sudo apt-get update
sudo apt-get upgrade
```


```
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt update

sudo apt install g++-11
sudo apt install gcc-11

```


export GIT_CURL_VERBOSE=1
github无法使用https拉取代码



* 单机docker换源，编辑/etc/docker/daemon.json，配置为
```
{
	"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
```
然后使用systemctl daemon-reload重新加载配置，使用systemctl restart docker重启docker即可


改变deployment的配置文件的镜像，他会自动重启然后拉取镜像启动服务，所以需要准备自己的镜像，一般只需要替换镜像中的一个或者几个文件，然后让k8s使用此镜像即可，但是无法直接替换镜像中的文件，需要在运行的容器中替换，然后再以此为基准建立新的镜像，然后替换

## mysql
* 安装

```
sudo apt install mysql-server
mysql_secure_installation
```

密码问题
```
/etc/mysql/mysql.conf.d/mysqld.cnf/
添加
skip-grant-tables
然后使用mysql进入mysql
主要修改user下的用户的密码字段
无法直接使用update语句，
可以使用ALTER USER 'root'@'localhost' IDENTIFIED BY '123456'; 修改
如果报权限问题，使用flush privileges;刷新在执行

之后再修改配置文件去掉添加的信息即可

密码策略，
SHOW VARIABLES LIKE 'validate_password%';查看密码策略
使用set global validate_password_length=6;修改策略


```





## Nodejs 升级

安装Slidev的时候，需要升级node
1. 清除缓存  
`sudo npm cache clean -f`
2. 安装版本管理工具  
`sudo npm install n -g`
3. 使用版本管理工具安装指定node或者升级到最新node版本  
`sudo n stable （安装node最新版本`

退出当前session，重新进入，node -v检查
