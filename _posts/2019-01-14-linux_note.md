---
layout:     post
title:      linux-note
subtitle:     linux-note
date:       2019-01-14
author:     hanlei
header-img: img/bg-linux.jpg
catalog: true
tags:
    - linux 
---

[TOC]
# linux-note
## kernel update
### yum 
```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum install -y kernel.x86_64
yum --enablerepo=elrepo-kernel install kernel-ml -y
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
grub2-editenv list
grub2-set-default "CentOS Linux (4.20.0-1.el7.elrepo.x86_64) 7 (Core)"
grub2-editenv list
reboot
uname -r
cat /boot/grub2/grub.cfg |grep menuentry
```
### 编译方式
```
wget http://10.89.8.200:8080/kernel/linux-4.19.1.tar.xz
tar -xf linux-4.19.1.tar.xz
cd linux-4.19.1/
cp /boot/config-3.10.0-327.el7.x86_64 .config
yum -y install ncurses-devel elfutils-libelf-devel
make menuconfig
make
make modules_install

ls /lib/modules
make install
ll -h /boot/
chmod 755 /boot/vmlinuz-4.19.1
cat /boot/grub2/grub.cfg |grep menuentry
vim /boot/grub2/grub.cfg
grub2-set-default "CentOS Linux (4.19.1) 7 (Core)"
grub2-editenv list 

reboot
uname -r

```
## iptables
```
iptables开放某个端口，写在drop前生效
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

放开某段端口
iptables -A INPUT -p tcp --dport 12389:12398 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 12389:12398 -j ACCEPT

端口转发
-A PREROUTING -s 1.1.1.1/32 -p tcp -m tcp --dport 12392 -j DNAT --to-destination 2.2.2.2:12392
-A POSTROUTING -s 2.2.2.2/32 -d 2.2.2.2/32 -p tcp -m tcp --dport 12392 -j MASQUERADE    
本地转发
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 3000

端口转发实例：
-A PREROUTING -s 1.1.1.1/32 -p tcp -m tcp --dport 7915 -j DNAT --to-destination 2.2.2.2:7915
-A PREROUTING -s 1.1.1.1/32 -p tcp -m tcp --dport 7895 -j DNAT --to-destination 2.2.2.2:7895
-A POSTROUTING -s 1.1.1.1/32 -d 2.2.2.2/32 -p tcp -m tcp --dport 7915 -j MASQUERADE
-A POSTROUTING -s 1.1.1.1/32 -d 2.2.2.2/32 -p tcp -m tcp --dport 7895 -j MASQUERADE

```
## elk
### es head
```

先安装node
wget https://nodejs.org/dist/v0.10.48/node-v0.10.48.tar.gz加压，make，make install
node --version
git clone https://github.com/mobz/elasticsearch-head.git
cd elasticsearch-head/
npm install 
```

所有依赖包安装成功后，修改 elasticsearch-head 目录下的 Gruntfile.js 文件，在 options 属性内增加 hostname，设置为 0.0.0.0。
```
connect: {
    server: {
        options: {
            hostname: '0.0.0.0',
            port: 9100,
            base: '.',
            keepalive: true
        }
    }
}
```

修改 Elasticsearch 配置文件 config/elasticsearch.yml在配置文件最后增加两个配置项，这样 elasticsearch-head 插件才可以访问 Elasticsearch 。
```

http.cors.enabled: true 
http.cors.allow-origin: "*"
```

npm run start
### nginx log
elk添加ip -map
logstash
```

input{
beats{
port => "5044"
}
}

filter{
#grok{match => { "message" => "%{IP:client_ip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] \"%{WORD:method} /%{NOTSPACE:request_page} HTTP/%{NUMBER:http_version}\" %{NUMBER:status}" }}
grok {match => {"message" => "%{IP:client_ip} %{USER:remote_user} %{USER:remote_auth} \[%{HTTPDATE:timestamp}\] %{QUOTEDSTRING:request} %{NUMBER:status_code} %{NUMBER:body_bytes_sent} %{QUOTEDSTRING:http_referer} %{QUOTEDSTRING:http_user_agent} %{QUOTEDSTRING:remote_addr} %{QUOTEDSTRING:upstream_response_time} %{QUOTEDSTRING:request_time}" }}
geoip{source => "client_ip"
target => "geoip"
database => "/usr/share/logstash/vendor/bundle/jruby/2.3.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-City.mmdb"
add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}" ]}
#
mutate {
convert => [ "[geoip][coordinates]", "float" ]
#convert => [ "[request_time]", "float" ]
#convert => [ "[upstream_response_time]", "float" ] 
#
}
}
output{
elasticsearch{
hosts => ["ip9200"]
index => "logstash-www-%{+YYYY.MM.dd}"
}
}
```
### 解析mongo日志

logstash配置
```

input {
beats {
port => "5044"
type => "mongodblog"
}
}

 

filter {
if [type] == "mongodblog" {
grok {
match => ["message","%{TIMESTAMP_ISO8601:timestamp}\s+I %{WORD:MONGO_ACTION}\s+\[%{WORD:SOCK_ACTION}\]\s+%{GREEDYDATA:body}"]
remove_field => [ "message" ]
}

if [body] =~ "ms$" { 
grok {
match => ["body","%{WORD:command_action}\s+%{WORD:dbname}\.\$?%{WORD:collname}\s+%{GREEDYDATA:command_content}\s+%{NUMBER:time_spend}ms"]
}
}

date {
match => [ "timestamp", "UNIX", "YYYY-MM-dd HH:mm:ss", "ISO8601"]
remove_field => [ "timestamp" ]
}

mutate {
remove_field => ["message"]
}
}
}

 

 

output {
elasticsearch {
hosts => ["http://127.0.0.1:9200"]
index => "mongo-%{+YYYY.MM.dd}"
}
}

```

参考日志格式
```

2018-03-06T03:11:51.338+0800 I COMMAND  [conn1978967] command top_fba.$cmd command: createIndexes { createIndexes: "top_amazon_fba_inventory_data_2018-03-06", indexes: [ { key: { sellerId: 1,
 sku: 1, updateTime: 1 }, name: "sellerId_1_sku_1_updateTime_1" } ] } keyUpdates:0 writeConflicts:0 numYields:0 reslen:113 locks:{ Global: { acquireCount: { r: 3, w: 3 } }, Database: { acquir
eCount: { w: 2, W: 1 } }, Collection: { acquireCount: { w: 1 } }, Metadata: { acquireCount: { w: 2 } }, oplog: { acquireCount: { w: 2 } } } protocol:op_query 5751ms
```


