Mesos+Marathon+docker集群部署
---
| 版本 | 作者 |    日期    | 备注 |
| ---- | ---- | ---------- | ---- |
|  1.0 | VF   | 2015-10-09 | 无   |
|  2.0 | VF   | 2015-10-15 | 无   |
##0.内容导航
1. 整体架构
2. 环境准备
3. Master节点安装
4. Slave节点安装
5. Mesos简单实用
6. Docker私有仓库
7. 负载均衡与服务发现

##1.整体架构
|     节点名称    | 节点类型 |      IP      |            组件            |
| --------------- | -------- | ------------ | -------------------------- |
| master101       | master   | 192.168.2.71 | mesos、marathon、zookpeer  |
| master102       | master   | 192.168.2.72 | mesos、marathon、zookpeer  |
| master103       | master   | 192.168.2.73 | mesos、marathon、zookpeer  |
| slave101        | slave    | 192.168.2.61 | mesos、docker              |
| slave102        | slave    | 192.168.2.62 | mesos、docker              |
| slave103        | slave    | 192.168.2.63 | mesos、docker              |
| bamboo101       | 负载均衡 | 192.168.2.91 | haproxy、bamboo、keeplived |
| bamboo102       | 负载均衡 | 192.168.2.92 | haproxy、bamboo、keeplived |
| bamboo103       | 负载均衡 | 192.168.2.93 | haproxy、bamboo、keeplived |
>说明：集群模式部署，master节点应该是奇数，最少为3个节点，便于leader选举

##2.环境准备
>操作系统：Centos7.1 Minimal  
>Mesos版本：0.24.1  
>Marathon版本：0.11.0  
>Docker版本：1.7.1

- 关闭selinux（重启）  
```bash
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
- 关闭防火墙  
```bash
systemctl disable firewalld.service
```
- 清空iptables  
```bash
iptables -F
```
- 升级centos包：
```bash
yum update
```
- 安装mesosphere仓库  
```bash
rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum clean all
yum makecache
```
>说明：以上部署所有节点执行

##3.Master节点安装
####组件安装
```bash
yum -y install mesos marathon mesosphere-zookeeper
```
####配置zookeeper
```bash
#master101
echo 1 > /var/lib/zookeeper/myid
#master102
echo 2 > /var/lib/zookeeper/myid
#master103
echo 3 > /var/lib/zookeeper/myid
```  
编辑文件/etc/zookeeper/conf/zoo.cfg，追加以下内容
```bash
server.1=192.168.2.71:2888:3888
server.2=192.168.2.72:2888:3888
server.3=192.168.2.73:2888:3888
```
重启zookeeper服务
```bash
systemctl start zookeeper
```
####配置mesos
配置mesos使用zookeeper
```bash
echo zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/mesos > /etc/mesos/zk
echo 2 > /etc/mesos-master/quorum
```
配置mesos-master组件的hostname和ip参数
master01
```bash
#master101
echo 192.168.2.71 > /etc/mesos-master/hostname
echo 192.168.2.71 > /etc/mesos-master/ip
#master102
echo 192.168.2.72 > /etc/mesos-master/hostname
echo 192.168.2.72 > /etc/mesos-master/ip
#master103
echo 192.168.2.73 > /etc/mesos-master/hostname
echo 192.168.2.73 > /etc/mesos-master/ip
```
> 说明：hostname可以不配置，默认使用机器名

停用masrer节点上的mesos-slave服务
```bash
systemctl stop mesos-slave.service && systemctl disable mesos-slave.service
```
重启mesos-master服务
```bash
systemctl restart mesos-master.service
```
####配置marathon
```bash
mkdir -p /etc/marathon/conf
```
直接复制mesos的hostname文件
```bash
cp /etc/mesos-master/hostname /etc/marathon/conf
```
####配置marathon使用zookeeper（可选）  
默认情况下marathon会自动获取本机的mesos的zk配置，并且会根据zookeeper的配置。为自己添加marathon的配置同步，手动添加以下配置文件是为了便于管理，同样，其他的marathon参数，都可以以参数名称命名一个文件，存放在/etc/marathon/conf目录下，然后在其中设置参数值的形式为marathon作进一步配置
提示：在本例中，默认的marathon启动命令为：
```bash
marathon: run_jar --hostname 192.168.2.71 --zk zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/marathon --master zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/mesos
```
第一步：连接mesos的zookeeper
```bash
cp /etc/mesos/zk /etc/marathon/conf/master
```
第二步：配置marathon使用的zookeeper
```bash
echo zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/marathon /etc/marathon/conf/zk
```  
####启动服务及检测
重启marathon服务
```bash
systemctl restart marathon
```
检查端口监听情况
>说明：如果没有lsof命令可以下载安装<br>
yum install -y lsof

zookeeper端口：lsof -i :2181 -n
```bash
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java      2893 root   23u  IPv6  23787      0t0  TCP *:eforward (LISTEN)
java      2893 root   27u  IPv6  25757      0t0  TCP 192.168.2.71:eforward->192.168.2.71:35808 (ESTABLISHED)
java      2893 root   30u  IPv6  24204      0t0  TCP 192.168.2.71:eforward->192.168.2.71:35812 (ESTABLISHED)
java      2893 root   31u  IPv6  25810      0t0  TCP 192.168.2.71:eforward->192.168.2.71:35852 (ESTABLISHED)
java      2893 root   32u  IPv6  25803      0t0  TCP 192.168.2.71:eforward->192.168.2.72:57387 (ESTABLISHED)
java      2957 root   15u  IPv4  25761      0t0  TCP 192.168.2.71:35812->192.168.2.71:eforward (ESTABLISHED)
java      2957 root   31u  IPv6  25157      0t0  TCP 192.168.2.71:35808->192.168.2.71:eforward (ESTABLISHED)
java      2957 root   37u  IPv6  25758      0t0  TCP 192.168.2.71:54372->192.168.2.72:eforward (ESTABLISHED)
mesos-mas 3342 root   17u  IPv4  25809      0t0  TCP 192.168.2.71:35852->192.168.2.71:eforward (ESTABLISHED)
mesos-mas 3342 root   18u  IPv4  26920      0t0  TCP 192.168.2.71:54419->192.168.2.72:eforward (ESTABLISHED)
mesos-mas 3342 root   22u  IPv4  25817      0t0  TCP 192.168.2.71:54420->192.168.2.72:eforward (ESTABLISHED)
mesos-mas 3342 root   23u  IPv4  25816      0t0  TCP 192.168.2.71:54418->192.168.2.72:eforward (ESTABLISHED)
```
mesos端口：lsof -i :5050 -n
```bash
java      2957 root   27u  IPv4  25326      0t0  TCP 192.168.2.71:52901->192.168.2.72:mmcc (ESTABLISHED)
mesos-mas 3342 root    5u  IPv4  24286      0t0  TCP 192.168.2.71:mmcc (LISTEN)
mesos-mas 3342 root   24u  IPv4  26921      0t0  TCP 192.168.2.71:52898->192.168.2.72:mmcc (ESTABLISHED)
mesos-mas 3342 root   25u  IPv4  26967      0t0  TCP 192.168.2.71:60935->192.168.2.73:mmcc (ESTABLISHED)
mesos-mas 3342 root   26u  IPv4  25818      0t0  TCP 192.168.2.71:mmcc->192.168.2.72:45845 (ESTABLISHED)
mesos-mas 3342 root   27u  IPv4  25823      0t0  TCP 192.168.2.71:mmcc->192.168.2.73:51978 (ESTABLISHED)
```
其他端口
```bash
mesos通信端口：lsof -i :2888 -n
mesos选举端口：lsof -i :3888 -n
marathon端口：lsof -i :8080 -n
```
服务重启命令
```bash
systemctl restart zookeeper
systemctl restart mesos-master
systemctl restart marathon
```
配置开机启动
```bash
chkconfig zookeeper on
chkconfig mesos-master on
chkconfig marathon on
```
##4.Salve节点安装
####组件安装
```bash
yum -y install mesos docker
```
####配置mesos，与master一致
```bash
echo zk://192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181/mesos > /etc/mesos/zk
```
####配置mesos-slave  
slave101
```bash
echo 192.168.2.61 > /etc/mesos-slave/hostname
echo 192.168.2.61 > /etc/mesos-slave/ip
```
slave102
```bash
echo 192.168.2.62 > /etc/mesos-slave/hostname
echo 192.168.2.62 > /etc/mesos-slave/ip
```
slave103
```bash
echo 192.168.2.63 > /etc/mesos-slave/hostname
echo 192.168.2.63 > /etc/mesos-slave/ip
```
>hostname可以不配置，默认使用机器名

####配置mesos-slave使用docker容器
```bash
echo 'docker,mesos' > /etc/mesos-slave/containerizers
echo '5mins' > /etc/mesos-slave/executor_registration_timeout
```
如果使用本地docker仓库，需要配置docker
```bash
sed -i "s/^OPTIONS='--selinux-enabled'/OPTIONS='--selinux-enabled --insecure-registry 192.168.2.98:5000'/g" /etc/sysconfig/docker
```
>说明：`192.168.2.98:5000`是本环境中部署的docker registry仓库地址

####启动服务
停用slave节点上的mesos-master服务
```bash
systemctl stop mesos-master.service && systemctl disable mesos-master.service
```
服务重启命令
```bash
systemctl restart docker
systemctl restart mesos-slave
```
配置开机启动
```bash
chkconfig docker on
chkconfig mesos-slave on
```
##5.Mesos简单使用
####Mesos控制台  
Mesos的控制台地址：`http://192.168.2.71:5050`
Mesos的控制台上可以查看的当前的资源实用情况、Slave节点状态、当前运行的Task、完成的Task、可以切换到Framework（如Marathon）或者Slave。
####Marathon控制台  
Marathon控制台地址：`http://192.168.2.71:8080`
Marathon控制台上可以查看当前应用的运行状态，可以发布新应用、调整当前应用的实例数等。  
发布应用：
```bash
ID：hello
CPUs：0.1
Memory：16
Disk Space：0
Instances：1
Command：echo hello world!;sleep 10;
```
上述例子是使用Mesos默认容器进行创建，并执行任务；如果需要使用docker容器，可以在`Docker container settings`中填写具体参数。
应用创建后，会输出`hello world!`，并且每10秒钟重新deploy一次。
>说明：Mesos+Marathon框架发布的Task会在执行完毕后进行销毁，并重新发布，若是需要发布web app一类的长应用，必须保证运行程序独占console，如Tomcat，可以执行`catalina.sh run`进行启动并保持长运行状态，后台运行的程序Marathon无法保持Task长运行。

可以通过Marathon的RestAPI发布应用
```bash
curl -X POST http://192.168.2.71:8080/v2/apps -d@inky.json -H "Content-type: application/json"
```
inky.json文件示例
```json
{
    "id": "inky4",
    "container": {
        "docker": {
            "image": "192.168.2.98:5000/tomcat-jdk1.7",
            "network":"BRIDGE",
            "portMappings":[{"containerPort":8080,"hostPort":0,"protocol": "tcp"}]
        },
        "type": "DOCKER",
        "volumes": []
    },
    "args": [],
    "cpus": 1,
    "mem": 512,
    "instances": 1
}
```

##6.Docker私有仓库
####仓库搭建
方式一：通过docker image构建  
`docker run -d -p 5000:5000 -v /opt/data/registry:/tmp/registry:rw registry`
>说明：`/tmp/registry`:容器中镜像的存储位置  
`/opt/data/registry`：挂载`/tmp/registry`到本地宿主机上的目录

方式二：通过本地安装docker-registry构建
```bash
yum install -y python-devel libevent-devel python-pip gcc xz-devel
python-pip install docker-registry
```
存储路径修改：  
`cp config/config_sample.yml config/config.yml`
修改其中的：`storage_path`参数。  
启动docker-registry的web服务  
`gunicorn -c contrib/gunicorn.py docker_registry.wsgi:application`
####上传、下载镜像
1. 从公有创库中获取基础镜像（docker pull）
2. 对基础镜像做出修改（docker commit）
3. 标记本地镜像（docker tag）
4. 推送镜像到私有仓库（docker push）
5. 获取私有仓库镜像（docker pull）

##7.负载均衡与服务发现
####Haproxy组件
安装
```bash
yum install -y haproxy
```
配置忽略VIP及开启转发
```bash
echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
echo net.ipv4.ip_nonlocal_bind=1 >> /etc/sysctl.conf
```
重启系统或者执行以下命令
```bash
sysctl -e net.ipv4.ip_forward=1
sysctl -e net.ipv4.ip_nonlocal_bind=1
```
重启haproxy服务
```bash
systemctl restart haproxy
```
####Bamboo组件
通过脚本自动安装
```bash
curl https://raw.githubusercontent.com/VFT/bamboo/master/install.sh | sh
mkdir /var/bamboo
cp /opt/bamboo/config/haproxy_template.cfg /var/bamboo/
cp /opt/bamboo/config/production.example.json /var/bamboo/production.json
```
编辑/var/bamboo/production.json内容如下：
```bash
{
  "Marathon": {
    "Endpoint": "http://192.168.2.71:8080,http://192.168.2.72:8080,http://192.168.2.73:8080"
  },

  "Bamboo": {
    "Host": "http://192.168.2.98:8000",
    "Zookeeper": {
      "Host": "192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181",
      "Path": "/marathon-haproxy/state",
      "ReportingDelay": 5
    }
  },
  "HAProxy": {
    "TemplatePath": "/var/bamboo/haproxy_template.cfg",
    "OutputPath": "/etc/haproxy/haproxy.cfg",
    "ReloadCommand": "haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -D -sf $(cat /var/run/haproxy.pid)",
    "ReloadValidationCommand": "haproxy -c -f {{.}}",
    "ReloadCleanupCommand": "exit 0"
  },
  "StatsD": {
    "Enabled": false,
    "Host": "localhost:8125",
    "Prefix": "bamboo-server.production."
  }
}
```
>说明：Marathon.Endpoint：Marathon服务的访问地址，Bamboo.Host：Bamboo服务控制台地址，Bamboo.Zookeeper.Host：Zookeeper服务访问地址

启动Bamboo服务
```bash
nohup /opt/bamboo/bamboo -haproxy_check -config="/var/bamboo/production.json" &
```
停止Bamboo服务
```bash
kill $(lsof -i:8000 |awk '{print $2}' | tail -n 2)
```
>说明：可以用命令`netstat -ntpl`或者`lsof -i :5050 -n`查看进程

####Bamboo组件（源码安装）
安装start-stop-daemon
```bash
yum install gcc
wget http://developer.axis.com/download/distribution/apps-sys-utils-start-stop-daemon-IR1_9_18-2.tar.gz
tar zxf apps-sys-utils-start-stop-daemon-IR1_9_18-2.tar.gz
cd apps/sys-utils/start-stop-daemon-IR1_9_18-2/
gcc start-stop-daemon.c -o start-stop-daemon
cp start-stop-daemon /usr/bin
```
下载及安装bamboo源码包
```bash
wget https://github.com/QubitProducts/bamboo/archive/v0.2.14.tar.gz
tar xzvf v0.2.14.tar.gz
cd bamboo-0.2.14/
cp builder/init.d-bamboo-server /etc/init.d/bamboo-server
chown root:root /etc/init.d/bamboo-server
chmod 755 /etc/init.d/bamboo-server
chkconfig bamboo-server on
```
配置bamboo
```bash
mkdir /var/bamboo
cp config/haproxy_template.cfg /var/bamboo/
cp config/production.example.json /var/bamboo/production.json
```
修改/var/bamboo/production.json
```json
{
  "Marathon": {
    "Endpoint": "http://192.168.2.71:8080,http://192.168.2.72:8080,http://192.168.2.73:8080"
  },

  "Bamboo": {
    "Host": "http://192.168.2.98:8000",
    "Zookeeper": {
      "Host": "192.168.2.71:2181,192.168.2.72:2181,192.168.2.73:2181",
      "Path": "/marathon-haproxy/state",
      "ReportingDelay": 5
    }
  },
  "HAProxy": {
    "TemplatePath": "/var/bamboo/haproxy_template.cfg",
    "OutputPath": "/etc/haproxy/haproxy.cfg",
    "ReloadCommand": "haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -D -sf $(cat /var/run/haproxy.pid)"
    "ReloadValidationCommand": "haproxy -c -f {{.}}",
    "ReloadCleanupCommand": "exit 0"
  },
  "StatsD": {
    "Enabled": false,
    "Host": "localhost:8125",
    "Prefix": "bamboo-server.development."
  }
}
```
启动Bamboo服务  
`systemctl start bamboo-server`  
停止Bamboo服务  
`systemctl start bamboo-server`  
####Bamboo简单使用
按上述配置Bamboo安装后以后，Bamboo监听`8000`端口，可以通过`http://<IP>:8000`来访问bamboo的控制台  
![bamboo console](https://raw.githubusercontent.com/VFT/imageStore/master/bamboo01.png)  
添加转发规则  
![bamboo edit](https://raw.githubusercontent.com/VFT/imageStore/master/bamboo02.png)  
现在，可以通过`http://<IP>`，默认端口：`80`来访问`inky1`这个app。
