---
layout:     post
title:      openstack
subtitle:   openstack
date:       2019-08-16
author:     hl
header-img: img/bg-k8s.jpg
catalog: true
tags:
    - openstack
---

[TOC]

# openstack 组件介绍

## keystone

### 介绍

[官方](https://docs.openstack.org/keystone)

Keystone is an OpenStack service that provides API client authentication, service discovery, and distributed multi-tenant authorization by implementing [OpenStack’s Identity API](https://docs.openstack.org/api-ref/identity/index.html).

### 常用操作

#### 版本一

OpenStack 身份认证服务 (Identity Service），即 Keystone，是为 OpenStack 云环境中用户的 账户和角色信息提供认证和管理服务的。 在 Keystone 里，我们有如下概念：租户、角色和用户。租户就像一个项目，它有一些资源，比如用户、镜像和实例，并且其中有仅仅对该项目可知的网络。 用户可隶属于一个或多 个租户，并且可以在这些项目中切换，去获取相应资源。租户里的用户可以被指定为多种角 色。在最基本的应用场景里，一个用户可以被指定为管理员角色，或者只是成员角色。当用 户在租户中拥有管理员特权时，他们可以使用那些影响租户的功能（比如修改外部网络）。 反之，一个普通用户被指定为成员角色，它通常被指定执行与用户相关的角色，比如旋转实 例、创建卷和创建租户唯一网络。

```
我们从创建一个名为 cookbook 的租户开始。
keystone tenant-create --name cookbook - -description ”Default cookbooK Tenant" --enabled true 

创建 adrnin 角色。
keystone role-create --name admin 

创建用户。
keystone user-create  --name user_namee  --tenant_id TENANT_ID  --pass PASSWORD  --email email_address  --enabled true 
赋予一个用户某个角色的命令
keystone user-role-add  --user USER_ID --role ROLE_ID  --tenant id TENANT_ID 

keystone tenant-list 
keystone role-list 
keystone user-list

通过 keystone 客户端的 service-create 选项来添加服务，
keystone service-create --name service_name  --type service_type  --description ’ description ’
定义各个 服务对应的端点。
keystone endpoint-create  --region region_name  --service_id service_id  --publicurl public_url  --adminurl admin_url  --internalurl internal url 
keystone service-list

```

#### 版本二

Keystone is organized as a group of internal services exposed on one or many endpoints. Many of these services are used in a combined fashion by the frontend. For example, an authenticate call will validate user/project credentials with the Identity service and, upon success, create and return a token with the Token service.[参考](https://docs.openstack.org/keystone/latest/getting-started/architecture.html)

[用户api操作文档v3](https://docs.openstack.org/keystone/latest/api_curl_examples.html)



```
openstack application credential create monitoring
openstack application credential create monitoring --secret securesecret
openstack application credential create monitoring --role Member
Create a user named alice:
openstack user create --password-prompt --email alice@example.com alice
Create a project named acme:
openstack project create acme --domain default
Create a domain named emea:
openstack --os-identity-api-version=3 domain create emea
Create a role named compute-user:
openstack role create compute-user
assign the compute-user role to the alice user in the acme project:
openstack role add --project acme --user alice compute-user
查看
openstack project list
openstack user list
openstack role list
openstack domain  list
openstack role show ROLE_NAME
创建
openstack project create --description 'my new project' new-project --domain default
openstack user create --project new-project --password PASSWORD new-user
openstack role create new-role
更新project
openstack project set PROJECT_ID --disable   #禁用
openstack project set PROJECT_ID --enable	#启用
openstack project set PROJECT_ID --name project-new    #改名
openstack project delete PROJECT_ID		#删除
#更新user
openstack user set USER_NAME --disable
openstack user set USER_NAME --enable
openstack user set USER_NAME --name user-new --email new-user@example.com
openstack user delete USER_NAME
#分发role
openstack role add --user USER_NAME --project TENANT_ID ROLE_NAME
openstack role add --user demo --project test-project new-role
openstack role assignment list --user USER_NAME --project PROJECT_ID --names   #验证
openstack role remove --user USER_NAME --project TENANT_ID ROLE_NAME   #删除
#服务相关
openstack service create --name SERVICE_NAME --description SERVICE_DESCRIPTION SERVICE_TYPE
openstack service create --name swift --description "object store service" object-store
openstack service show SERVICE_TYPE|SERVICE_NAME|SERVICE_ID
openstack service show object-store
openstack endpoint create nova public http://example.com/compute/v2.1
openstack service delete SERVICE_TYPE|SERVICE_NAME|SERVICE_ID
openstack service delete object-store
openstack project create service --domain default
openstack user create nova --password Sekr3tPass
openstack role add --project service --user nova admin
```



### 安装

[参考](https://docs.openstack.org/keystone/latest/index.html)

```
参考 https://docs.openstack.org/keystone/latest/install/keystone-install-rdo.html

1,添加openstack yum源：我这里使用163
再 /etc/yum.repos.d/CentOS7-Base-163.repo中添加（根据实际情况添加）
[openstack]
name=openstack
baseurl=http://mirrors.163.com/centos/7.6.1810/cloud/x86_64/openstack-queens/
gpgcheck=0
enabled=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7

或者 yum install -y centos-release-openstack-queens.noarch     #选择对应的版本
2，yum install -y openstack-keystone httpd mod_wsgi memcached python-memcached

3，数据库配置
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone';
flush privileges;
4，配置文件修改
vim /etc/hosts
127.0.0.1	localhos  controller

vim /etc/keystone/keystone.conf     
[DEFAULT]
admin_token = admin
log_file = /data/log/keystone/keystone.log      #注意权限
[database]
connection = mysql+pymysql://keystone:keystone@controller/keystone
[token]
provider = fernet

5，keystone-manage db_sync  #数据库填充
安装openstack客户端
yum install python-devel python-pip -y 
pip install python-openstackclient
pip install python-PROJECTclient
yum install python-PROJECTclient

6， 初始化密钥库
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
7，引导生成验证服务（就是生成uri）
keystone-manage bootstrap --bootstrap-password admin --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne
8，httpd
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
环境变量
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
在这里遇到个坑：如果在/etc/keystone/keystone.conf  出现的文件keystone用户没有权限。会出现500状态码。
9.熟悉domains, projects, users, and roles过程
openstack domain create --description "An Example Domain" example

openstack project create --domain default --description "Demo Project" myproject
openstack user create --domain default --password-prompt myuser
openstack role create myrole
openstack role add --project myproject --user myuser myrole

```



## swift

[官方](https://docs.openstack.org/swift/latest/)

### 介绍

OpenStack Swift 开源项目提供了弹性可伸缩、高可用的分布式对象存储服务，适合存储大规模非结构化数据。 OpenStack Object Storage (Swift)是OpenStack开源云计算项目的子项目之一。Swift的目的是使用普通硬件来构建冗余的、可扩展的分布式对象存储集群，存储容量可达PB级。Swift并不是文件系统或者实时的数据存储系统，它是对象存储，用于永久类型的静态数据的长期存储，这些数据可以检索、调整，必要时进行更新。最适合存储的数据类型的例子是虚拟机镜像、图片存储、邮件存储和存档备份。Swift无需采用RAID（磁盘冗余阵列），也没有中心单元或主控结点。Swift通过在软件层面引入一致性哈希技术和数据冗余性，牺牲一定程度的数据一致性来达到高可用性（High Availability，简称HA）和可伸缩性，支持多租户模式、容器和对象读写操作，适合解决互联网的应用场景下非结构化数据存储问题。

代理服务(ProxyServer)：Swift通过Proxy Server向外提供基于HTTP的REST服务接口，会根据环的信息来查找服务地址并转发用户请求至相应的账户、容器或者对象，进行CRUD(增删改查)等操作。由于采用无状态的REST请求协议，可以进行横向扩展来均衡负载。在访问Swift服务之前，需要先通过认证服务获取访问令牌，然后在发送的请求中加入头部信息 X-Auth-Token。代理服务器负责Swift架构的其余组件间的相互通信。代理服务器也处理大量的失败请求。例如，如果对于某个对象PUT请求时，某个存储节点不可用，它将会查询环可传送的服务器并转发请求。对象以流的形式到达(来自) 对象服务器，它们直接从代理服务器传送到(来自)用户—代理服务器并不缓冲它们。
认证服务(AuthenticationServer)：验证访问用户的身份信息，并获得一个对象访问令牌(Token)，在一定的时间内会一直有效；验证访问令牌的有效性并缓存下来直至过期时间。
缓存服务(CacheServer)：缓存的内容包括对象服务令牌，账户和容器的存在信息，但不会缓存对象本身的数据；缓存服务可采用Memcached集群，Swift会使用一致性哈希算法来分配缓存地址。
账户服务(AccountServer)：提供账户元数据和统计信息，并维护所含容器列表的服务，每个账户的信息被存储在一个SQLite数据库中。
容器服务(ContainerServer)：提供容器元数据和统计信息(比如对象的总数，容器的使用情况等)，并维护所含对象列表的服务。容器服务并不知道对象存在哪，只知道指定容器里存的哪些对象。 这些对象信息以SQLite数据库文件的形式存储，和对象一样在集群上做类似的备份。 
对象服务(ObjectServer)：提供对象元数据和内容服务，可以用来存储、检索和删除本地设备上的对象。在文件系统中，对象以二进制文件的形式存储，它的元数据存储在文件系统的扩展属性(xattr)中，建议采用默认支持扩展属性(xattr)的XFS文件系统。每个对象使用对象名称的哈希值和操作的时间戳组成的路径来存储。最后一次写操作总可以成功，并确保最新一次的对象版本将会被处理。删除也被视为文件的一个版本(一个以".ts"结尾的0字节文件，ts表示墓碑)。
复制服务(Replicator)：会检测本地分区副本和远程副本是否一致，具体是通过对比哈希文件和高级水印来完成，发现不一致时会采用推式(Push)更新远程副本：对于对象的复制，更新只是使用rsync同步文件到对等节点。帐号和容器的复制通过HTTP或rsync来推送整个数据库文件上丢失的记录；另外一个任务是确保被标记删除的对象从文件系统中移除：当有一项(对象、容器、或者帐号)被删除，则一个墓碑文件被设置作为该项的最新版本。复制器将会检测到该墓碑文件并确保将它从整个系统中移除。
更新服务(Updater)：当对象由于高负载或者系统故障等原因而无法立即更新时，任务将会被序列化到在本地文件系统中进行排队，以便服务恢复后进行异步更新；例如成功创建对象后容器服务器没有及时更新对象列表，这个时候容器的更新操作就会进入排队中，更新服务会在系统恢复正常后扫描队列并进行相应的更新处理。
审计服务(Auditor)：在本地服务器上会反复地爬取来检查对象，容器和账户的完整性，如果发现比特级的错误，文件将被隔离，并复制其他的副本以覆盖本地损坏的副本；其他类型的错误(比如在任何一个容器服务器中都找不到所需的对象列表)会被记录到日志中。
账户清理服务(AccountReaper)：移除被标记为删除的账户，删除其所包含的所有容器和对象。删除账号的过程是相当直接的。对于每个账号中的容器，每个对象先被删除然后容器被删除。任何失败的删除请求将不会阻止整个过程，但是将会导致整个过程最终失败(例如，如果一个对象的删除超时，容器将不能被删除，因此账号也不能被删除)。整个处理过程即使遭遇失败也继续执行，这样它不会因为一个麻烦的问题而中止恢复集群空间。账号收割器将会继续不断地尝试删除账号直到它最终变为空，此时数据库在db_replicator中回收处理，最终移除这个数据库文件。



### 安装

1修改host文件

```
vim /etc/hosts
10.21.8.88 controller proxy88
10.21.8.89 proxy89
10.21.8.90 object90
10.21.8.91 object91
10.21.8.92 object92

```

2 为swift创建用户

```
openstack user create --domain default --password-prompt swift    #user
openstack role add --project service --user swift admin			 #role
openstack service create --name swift --description "OpenStack Object Storage" object-store   #service
openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(project_id\)s      #api-endpoints public
openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s		# api-endpoints internal
openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1    #admin
```

3 安装proxy(controller)

```
yum install openstack-swift-proxy python-swiftclient python-keystoneclient python-keystonemiddleware memcached
```

4 [获取配置文件并修改](https://docs.openstack.org/swift/latest/install/controller-install-rdo.html)

```
curl -o /etc/swift/proxy-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/proxy-server.conf-sample
```

5安装存储组件

```
1磁盘准备
yum install xfsprogs rsync
mkfs.xfs /dev/sdb
mkfs.xfs /dev/sdc
mkdir -p /srv/node/sdb
mkdir -p /srv/node/sdc

vim /etc/fstab
/dev/sdb /srv/node/sdb xfs noatime,nodiratime,logbufs=8 0 2
/dev/sdc /srv/node/sdc xfs noatime,nodiratime,logbufs=8 0 2
mount /srv/node/sdb
mount /srv/node/sdc

2开启rsync服务
vim /etc/rsyncd.conf
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_INTERFACE_IP_ADDRESS    #改为相应的ip
[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock
[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock
[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock

systemctl enable rsyncd.service
systemctl start rsyncd.service

3 安装
yum install openstack-swift-account openstack-swift-container openstack-swift-object
4下载配置并修改
curl -o /etc/swift/account-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/account-server.conf-sample
curl -o /etc/swift/container-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/container-server.conf-sample
curl -o /etc/swift/object-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/object-server.conf-sample

5权限管理
chown -R swift:swift /srv/node
mkdir -p /var/cache/swift
chown -R root:swift /var/cache/swift
chmod -R 775 /var/cache/swift

```



6 创建ring

```
1创建account ring
cd /etc/swift
swift-ring-builder account.builder create 10 3 1
添加节点到ring
swift-ring-builder account.builder add --region 1 --zone 1 --ip 10.21.8.90/91/92 --port 6202 --device  sdb/sdc --weight 100

swift-ring-builder account.builder
重新平衡
swift-ring-builder account.builder rebalance

2创建container ring
cd /etc/swift
swift-ring-builder container.builder create 10 3 1
添加节点到ring
swift-ring-builder container.builder add --region 1 --zone 1 --ip 10.21.8.90/91/92 --port 6201 --device sdb/sdc --weight 100

swift-ring-builder container.builder
swift-ring-builder container.builder rebalance

3创建object ring
cd /etc/swift
swift-ring-builder object.builder create 10 3 1
swift-ring-builder object.builder add  --region 1 --zone 1 --ip 10.21.8.90/91/92 --port 6200 --device sdb/sdc --weight 100

swift-ring-builder object.builder
swift-ring-builder object.builder rebalance

```

7 分发ring配置文件

```
cd /etc/swift
scp account.ring.gz  10.21.8.{88-92}:/etc/swift
scp container.ring.gz  10.21.8.{88-92}:/etc/swift
scp object.ring.gz 10.21.8.{88-92}:/etc/swift
```

8 [修改swift配置](https://docs.openstack.org/swift/latest/install/finalize-installation-rdo.html)

```
curl -o /etc/swift/swift.conf https://opendev.org/openstack/swift/raw/branch/master/etc/swift.conf-sample
vim /etc/swift/swift.conf

chown -R root:swift /etc/swift
```



9启动各个服务

```
systemctl enable openstack-swift-proxy.service memcached.service
systemctl start openstack-swift-proxy.service memcached.service

systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl start openstack-swift-account.service openstack-swift-account-auditor.service openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl enable openstack-swift-container.service openstack-swift-container-auditor.service openstack-swift-container-replicator.service openstack-swift-container-updater.service
systemctl start openstack-swift-container.service openstack-swift-container-auditor.service openstack-swift-container-replicator.service openstack-swift-container-updater.service
systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service openstack-swift-object-replicator.service openstack-swift-object-updater.service
systemctl start openstack-swift-object.service openstack-swift-object-auditor.service openstack-swift-object-replicator.service openstack-swift-object-updater.service
```

10 验证

```
 chcon -R system_u:object_r:swift_data_t:s0 /srv/node
 . demo-openrc
 swift stat
 openstack container create container1
 openstack object create container1 FILE
 openstack object list container1
 openstack object save container1 FILE
```
