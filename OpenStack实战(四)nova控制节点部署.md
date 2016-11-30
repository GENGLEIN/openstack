# 一 nova 官方简介
---
###使用OpenStack计算服务来托管和管理云计算系统。OpenStack计算服务是基础设施即服务(IaaS)系统的主要部分，模块主要由Python实现。OpenStack计算组件请求OpenStack Identity服务进行认证；请求OpenStack Image服务提供磁盘镜像；为OpenStack dashboard提供用户与管理员接口。磁盘镜像访问限制在项目与用户上；配额以每个项目进行设定（例如，每个项目下可以创建多少实例）。OpenStack组件可以在标准硬件上水平大规模扩展，并且下载磁盘镜像启动虚拟机实例。

#### OpenStack计算服务由下列组件所构成：

####nova-api 服务
####接收和响应来自最终用户的计算API请求。此服务支持OpenStack计算服务API，Amazon EC2 API，以及特殊的管理API用于赋予用户做一些管理的操作。它会强制实施一些规则，发起多数的编排活动，例如运行一个实例。

####nova-api-metadata 服务
####接受来自虚拟机发送的元数据请求。``nova-api-metadata``服务一般在安装``nova-network``服务的多主机模式下使用。更详细的信息，请参考OpenStack管理员手册中的链接`Metadata service <http://docs.openstack.org/admin-guide/compute-networking-nova.html#metadata-service>`__ in the OpenStack Administrator Guide。


# 二 安装 nova先决条件
---
### 2.1 在安装和配置 Compute 服务前，你必须创建数据库:
<pre>
create database nova;
create database nova_api;
</pre>
### 2.2 对数据库进行正确的授权：  
<pre>
grant all on nova.* to 'nova'@'localhost' identified by 'nova';
grant all on nova.* to 'nova'@'%' identified by 'nova';         
grant all on nova_api.* to 'nova'@'localhost' identified by 'nova';
grant all on nova_api.* to 'nova'@'%' identified by 'nova';         
</pre>

# 三 nova 部署安装    
---
## 3.0 Nova相关名词介绍:
>API:负责接收和响应外部请求。支持OpenStackAPI     
>Cert:负责身份认证。       
>Scheduler:用于云主机调度。         
>Conductor:计算节点访问数据的中间件。     
>Consoleauth:用于控制台的授权验证。     
>Novncproxy:VNC代理    
>nova service-list #查看服务

### 3.1 安装软件包
<pre>
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler
# openstack-nova-api 负责接收和响应外部请求，支持OpenStack API,Ec2API
# openstack-nova-scheduler 用于云主机调度
# openstack-nova-conductor 计算节点访问数据库的中间件
# openstack-nova-console 用于控制台的授权验证
# openstack-nova-novncproxy VNC代理
</pre>

### 3.2 编辑/etc/nova/nova.conf文件并完成下面的操作
<pre>
#在``[api_database]``和``[database]``部分，配置数据库的连接：
3105 [database]
3128 connection=mysql+pymysql://nova:nova@192.168.56.11/nova

2160 [api_database]
2168 connection=mysql+pymysql://nova:nova@192.168.56.11/nova_api
</pre>

### 3.3 同步Compute 数据库：
<pre>
# su -s /bin/sh -c "nova-manage api_db sync" nova
# su -s /bin/sh -c "nova-manage db sync" nova
#查看同步数据
mysql -h 192.168.56.11 -unova -pnova -e "use nova;show tables"
+--------------------------------------------+
| Tables_in_nova                             |
+--------------------------------------------+
| agent_builds                               |
| aggregate_hosts                            |
| aggregate_metadata                         |
| aggregates                                 |
| allocations                                |
| block_device_mapping                       |
| bw_usage_cache                             |
| cells                                      |
| certificates                               |
# mysql -h 192.168.56.11 -unova -pnova -e "use nova_api;show tables"         
+--------------------+
| Tables_in_nova_api |
+--------------------+
| build_requests     |
| cell_mappings      |
| flavor_extra_specs |
| flavor_projects    |
| flavors            |
| host_mappings      |
| instance_mappings  |
| migrate_version    |
| request_specs      |
</pre>

### 3.4 认证keystone
<pre>
3522 [keystone_authtoken]
3523 auth_uri = http://192.168.56.11:5000
3524 auth_url = http://192.168.56.11:35357
3525 memcached_servers = 192.168.56.11:11211
3526 auth_type = password
3527 project_domain_name = default
3528 user_domain_name = default
3529 project_name = service
3530 username = nova
3531 password = nova
--------------------------------------------
 382 auth_strategy=keystone #使用keystone来认证

</pre>

### 3.5 配置rabbitmq
<pre>
# vim /etc/nova/nova.conf
2119 rpc_backend=rabbit
4458 rabbit_host=192.168.56.11
4464 rabbit_port=5672
4476 rabbit_userid=openstack
4480 rabbit_password=openstack
</pre>

### 3.6 编辑/etc/nova/nova.conf设置启用的api
<pre>
[DEFAULT]
enabled_apis=osapi_compute,metadata
1684 use_neutron=true
1561 firewall_driver=nova.virt.firewall.NoopFirewallDriver #关闭防火墙
</pre>

### 3.6 在``[vnc]``部分，配置VNC代理使用控制节点的管理接口IP地址 
<pre>
5427 vncserver_listen=192.168.56.11 
5451 vncserver_proxyclient_address=192.168.56.11
[glance]
3354 api_servers=http://192.168.56.11:9292 #配置镜像服务 API 的位置
4307 lock_path=/var/lib/nova/tmp #在 [oslo_concurrency] 部分，配置锁路径
</pre>

### 3.7 启动nova并配置开机启动
<pre>
# systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
# systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

</pre>

### 3.8 注册nova服务,在keystone上
<pre>
# 创建 nova 服务实体：
# source admin-openstack.sh 
# openstack service create --name nova \
>   --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 91dad9443c5b4209a366000c2acef785 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
#创建 Compute 服务 API 端点
# openstack endpoint create --region RegionOne \
>   compute public http://192.168.56.11:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 3d6cef1f6f3a45348e3739880674d59c             |
| interface    | public                                       |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 91dad9443c5b4209a366000c2acef785             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.56.11:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+

# openstack endpoint create --region RegionOne   compute admin http://192.168.56.11:8774/v2.1/%\(tenant_id\)s 
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | f72ae859322e47a7a7db6fbe1a6056ce             |
| interface    | admin                                        |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 91dad9443c5b4209a366000c2acef785             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.56.11:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+

# openstack endpoint create --region RegionOne   compute internal http://192.168.56.11:8774/v2.1/%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | ff9fea24370e4e988adb7295118ed13d             |
| interface    | internal                                     |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 91dad9443c5b4209a366000c2acef785             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://192.168.56.11:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+
#验证
# openstack host list
+-------------------------+-------------+----------+
| Host Name               | Service     | Zone     |
+-------------------------+-------------+----------+
| linux-node1.example.com | consoleauth | internal |
| linux-node1.example.com | conductor   | internal |
| linux-node1.example.com | scheduler   | internal |
+-------------------------+-------------+----------+
</pre>
### 四 nova 图解说明:
---

![nova说明](images/novab.jpg)
![nova说明](images/novad.jpg)
![nova说明](images/novae.jpg)
![nova说明](images/novag.jpg)
![nova说明](images/novah.jpg)