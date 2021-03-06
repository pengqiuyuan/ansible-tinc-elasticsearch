## About

This repository was produced by following the [How To Use Ansible to Set Up a Production Elasticsearch Cluster] .

#
## Elasticsearch 集群方案

* [x] 使用分布式配置管理工具`ansible`来做集群的部署

* [x] 硬件配置

* [x] 集群角色划分和隔离、 副本分片建议（控制`shard`数量）、容量的规划

* [x] 映射与模板

* [x] `Elasticsearch` 配置

* [x] `System` 配置

* [x] 安全机制

* [x] `Reindex`（重新索引）

* [x] 分词

* [x] 旧集群（状态、节点状态、集群节点）

**使用分布式配置管理工具 **`ansible`** 来做集群的部署，一句话，对于集群的初始部署，配置批量更改，集群版本升级，重启故障结点都会快捷和安全许多。**

`Ubuntu 16.04`

`101.201.101.192`

`101.201.143.135`

`101.201.142.148`

在 `101.201.101.192` 上安装 `ansible`

```
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

安装 `git`

`$ sudo apt-get install git`

克隆 `ansible-tinc-elasticsearch`此项目 `hosts、platbook` 文件已配置好，已下是解释：

```
$ git clone https://github.com/pengqiuyuan/ansible-tinc-elasticsearch.git
```

项目在`/home/dev`下，`/home/dev/ansible-tinc-elasticsearch/hosts` 文件：

```
cd /home/dev/ansible-tinc-elasticsearch
vi ./hosts
```

保存 `hosts`，三台机器 `vpn_ip` 为内网`ip`，`ansible_host`为公网`ip`

```
[vpn]
node01 vpn_ip=10.174.10.35 ansible_host=101.201.101.192
node02 vpn_ip=10.45.139.80 ansible_host=101.201.143.135
node03 vpn_ip=10.45.137.79 ansible_host=101.201.142.148
[removevpn]

[elasticsearch_master_nodes]
node01

[elasticsearch_master_data_nodes]
node03

[elasticsearch_client_nodes]
node02
```

`ansible` 配置私钥用来免密码登录：

```
sudo ssh-keygen
ssh-copy-id root@101.201.101.192
ssh-copy-id root@101.201.143.135
ssh-copy-id root@101.201.142.148
```

执行命令`ansible all -m ping`结果如下：

```
root@iZ2zedqkczsim0buihk3zoZ:/home/dev/ansible-tinc-elasticsearch# ansible all -m ping
node02 | SUCCESS => {
"changed": false,
"ping": "pong"
}
node01 | SUCCESS => {
"changed": false,
"ping": "pong"
}
node03 | SUCCESS => {
"changed": false,
"ping": "pong"
}
```

`sity.yml` 文件

```
---

- hosts: vpn
roles:
- tinc

- hosts: removevpn
roles:
- tinc-remove
```

执行命令`ansible-playbook site.yml`成功结果：

```
PLAY RECAP *********************************************************************
node01 : ok=18 changed=15 unreachable=0 failed=0
node02 : ok=18 changed=15 unreachable=0 failed=0
node03 : ok=18 changed=15 unreachable=0 failed=0ik 插件
```

安装`ansible`的主机`192`执行如下命令，修改系统配置（`System` 配置后面介绍），`/home/dev/ansible-tinc-elasticsearch`

```
ansible all -m raw -a '
grep "* - nofile 512000" /etc/security/limits.conf || echo "* - nofile 512000" >> /etc/security/limits.conf
grep "elasticsearch - nproc unlimited" /etc/security/limits.conf || echo "elasticsearch - nproc unlimited" >> /etc/security/limits.conf
grep "fs.file-max = 1024000" /etc/sysctl.conf || echo "fs.file-max = 1024000" >> /etc/sysctl.conf
grep "vm.max_map_count = 262144" /etc/sysctl.conf || echo "vm.max_map_count = 262144" >> /etc/sysctl.conf
sysctl -p'
```

成功如下：

```
root@iZ2zedqkczsim0buihk3zoZ:/home/dev/ansible-tinc-elasticsearch# ansible all -m raw -a '
> grep "* - nofile 512000" /etc/security/limits.conf || echo "* - nofile 512000" >> /etc/security/limits.conf
> grep "elasticsearch - nproc unlimited" /etc/security/limits.conf || echo "elasticsearch - nproc unlimited" >> /etc/security/limits.conf
> grep "fs.file-max = 1024000" /etc/sysctl.conf || echo "fs.file-max = 1024000" >> /etc/sysctl.conf
> grep "vm.max_map_count = 262144" /etc/sysctl.conf || echo "vm.max_map_count = 262144" >> /etc/sysctl.conf
> sysctl -p'
node01 | SUCCESS | rc=0 >>
* - nofile 512000
elasticsearch - nproc unlimited
fs.file-max = 1024000
vm.max_map_count = 262144
vm.swappiness = 0
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
fs.file-max = 1024000
vm.max_map_count = 262144
Shared connection to 101.201.101.192 closed.


node02 | SUCCESS | rc=0 >>
* - nofile 512000
省略
Shared connection to 101.201.143.135 closed.


node03 | SUCCESS | rc=0 >>
* - nofile 512000
省略
Shared connection to 101.201.142.148 closed.
```

`elasticsearch.yml` 文件，`ansible playbook` 用来完成集群的初始部署、配置

```
- hosts: elasticsearch_client_nodes
roles:
- { role: elasticsearch, es_instance_name: "node",
es_config: {
cluster.name: "test-cluster",
discovery.zen.ping.unicast.hosts: "node01, node02, node03",
network.host: "_tun0_, _local_",
http.port: 9222,
transport.tcp.port: 9333,
node.data: false,
node.master: false,
bootstrap.memory_lock: true,
discovery.zen.minimum_master_nodes: 2,
gateway.recover_after_nodes: 2,
action.destructive_requires_name: true,
indices.breaker.total.limit: 70%,
indices.breaker.fielddata.limit: 25%,
indices.breaker.request.limit: 40%,
indices.fielddata.cache.size: 20%,
indices.queries.cache.size: 40%,
indices.memory.index_buffer_size: 10%
}
}
vars:
es_templates: false
es_scripts: true
es_java_install: true
es_major_version: "5.x"
es_version: "5.2.2"
es_heap_size: "2g"
es_api_port: 9222
es_max_map_count: 262144
es_max_open_files: 512000

- hosts: elasticsearch_master_data_nodes
roles:
- { role: elasticsearch, es_instance_name: "node",
es_config: {
cluster.name: "test-cluster",
discovery.zen.ping.unicast.hosts: "node01, node02, node03",
network.host: "_tun0_, _local_",
http.port: 9222,
transport.tcp.port: 9333,
node.data: true,
node.master: true,
bootstrap.memory_lock: true,
discovery.zen.minimum_master_nodes: 2,
gateway.recover_after_nodes: 2,
action.destructive_requires_name: true,
indices.breaker.total.limit: 70%,
indices.breaker.fielddata.limit: 25%,
indices.breaker.request.limit: 40%,
indices.fielddata.cache.size: 20%,
indices.queries.cache.size: 40%,
indices.memory.index_buffer_size: 10%
}
}
vars:
es_templates: false
es_scripts: true
es_java_install: true
es_major_version: "5.x"
es_version: "5.2.2"
es_heap_size: "2g"
es_api_port: 9222
es_max_map_count: 262144
es_max_open_files: 512000

- hosts: elasticsearch_master_nodes
roles:
- { role: elasticsearch, es_instance_name: "node",
es_config: {
cluster.name: "test-cluster",
discovery.zen.ping.unicast.hosts: "node01, node02, node03",
network.host: "_tun0_, _local_",
http.port: 9222,
transport.tcp.port: 9333,
node.data: false,
node.master: true,
bootstrap.memory_lock: true,
discovery.zen.minimum_master_nodes: 2,
gateway.recover_after_nodes: 2,
action.destructive_requires_name: true,
indices.breaker.total.limit: 70%,
indices.breaker.fielddata.limit: 25%,
indices.breaker.request.limit: 40%,
indices.fielddata.cache.size: 20%,
indices.queries.cache.size: 40%,
indices.memory.index_buffer_size: 10%
}
}
vars:
es_templates: false
es_scripts: true
es_java_install: true
es_major_version: "5.x"
es_version: "5.2.2"
es_heap_size: "2g"
es_api_port: 9222
es_max_map_count: 262144
es_max_open_files: 512000
```

执行 `ansible-playbook elasticsearch.yml`，成功如下：

```
TASK [elasticsearch : Update Native Roles] *************************************

PLAY RECAP *********************************************************************
node01 : ok=48 changed=0 unreachable=0 failed=0
node02 : ok=49 changed=7 unreachable=0 failed=0
node03 : ok=46 changed=1 unreachable=0 failed=0
```

`elasticsearch` 实例的`restart、start、stop`，`node_elasticsearch.service`中的`node`为上面`playbook`中定义的名称`es_instance_name: "node"`

```
systemctl restart node_elasticsearch.service
systemctl start node_elasticsearch.service
systemctl stop node_elasticsearch.service
systemctl status node_elasticsearch.service
```

```
service node_elasticsearch restart
service node_elasticsearch start
service node_elasticsearch stop
service node_elasticsearch status
```

版本升级修改`playbook` 中 es\_version: "5.2.2" =&gt; es\_version: "5.3.0"，集群升级

**硬件配置**

`15 台 Ubuntu 16.04 8核 32G SSD`

`30 台 Ubuntu 16.04 8核 32G SSD`

`90 台 Ubuntu 16.04 8核 32G SSD`

**集群角色划分和隔离、 副本分片建议（控制**`shard`**数量）、容量的规划**

**映射与模板**

`Elasticsearch`** 配置**

`System`** 配置**

**安全机制**

修改 elasticsearch http（9222） 和 transport （9333）默认端口

nginx 安装 （某个client 节点安装，ufw 开启80、22端口）

sudo apt-get -y install nginx

/etc/nginx 目录下，

启动：nginx

检查：nginx -t

修改配置重启：nginx -s reload

查看nginx日志：/var/log/nginx

修改配置目录：/etc/nginx/sites-available/default 文件

```
location /port/{
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_redirect off;
proxy_pass http://127.0.0.1:9222/;
}
```

防火墙 ufw \(所有节点开启，只开启22端口\)

ufw enable\|status\|disable

ufw allow 80

ufw delete 80

ufw reload 生效，有延时

`Reindex`**（重新索引）**

分词

```
curl -XPOST 'http://node01:9200/index1/_analyze?analyzer=ik_max_word&text=%e4%b8%ad%e5%8d%8e%e4%ba%ba%e6%b0%91%e5%85%b1%e5%92%8c%e5%9b%bd%e5%9b%bd%e6%ad%8c&pretty'
```

```
curl -XPOST 'http://node01:9200/index1/_analyze?analyzer=ik_smart&text=%e4%b8%ad%e5%8d%8e%e4%ba%ba%e6%b0%91%e5%85%b1%e5%92%8c%e5%9b%bd%e5%9b%bd%e6%ad%8c&pretty'
```









