# 一 neutron简介







# 二 安装neutron先决条件  






# 三 neutron部署安装

[网络选项公共网络](http://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/neutron-controller-install-option1.html)   
[网络选项私有网络](http://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/neutron-controller-install-option2.html)

### 3.1 安装软件包(公共网络):
<pre>
# 控制节点
# yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
</pre>

### 3.2 修改数据库连接地址:
<pre>
# vim /etc/neutron/neutron.conf
684 connection = mysql+pymysql://neutron:neutron@192.168.56.11/neutron
</pre>

### 3.3 配置keystone认证服务访问:
<pre>
 761 [keystone_authtoken]
 762 auth_uri = http://192.168.56.11:5000
 763 auth_url = http://192.168.56.11:35357
 764 memcached_servers = 192.168.56.11:11211
 765 auth_type = password
 766 project_domain_name = default
 767 user_domain_name = default
 768 project_name = service
 769 username = neutron
 770 password = neutron
 27 auth_strategy = keystone #告诉他使用keystone
</pre>

### 3.4 配置 “RabbitMQ” 消息队列的连接：
<pre>
511 rpc_backend = rabbit
1203 rabbit_host = 192.168.56.11
1221 rabbit_userid = openstack
1225 rabbit_password = openstack
</pre>

### 3.5 在``[DEFAULT]``部分，启用ML2插件并禁用其他插件：
<pre>
30 core_plugin = ml2   #启用ML2插件
33 service_plugins =   #禁用其他插件
</pre>

### 3.6 配置nova部分，配置网络服务来通知计算节点的网络拓扑变化：
<pre>
137 notify_nova_on_port_status_changes = true #端口发生变化，通知nova
141 notify_nova_on_port_data_changes = true

938 [nova] #使用nova的用户连接做一些工作
auth_url = http://192.168.56.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova
-------------------------------------
1060 lock_path = /var/lib/neutron/tmp #配置锁路径
</pre>

### 3.7 配置 Modular Layer 2 (ML2) 插件 （二层网络）
<pre>
# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
107 type_drivers = flat,vlan,gre,vxlan,geneve #驱动的选择
116 mechanism_drivers = linuxbridge,openvswitch #启用Linuxbridge机制
112 tenant_network_types = #禁用私有网络
121 extension_drivers = port_security #启用端口安全扩展驱动

[ml2_type_flat]
153 flat_networks = public  #配置公共虚拟网络络
230 enable_ipset = true #启用 ipset 增加安全组规则的高效性
</pre>

### 3.8 配置Linuxbridge代理
>Linuxbridge代理为实例建立layer－2虚拟网络并且处理安全组规则。
<pre>
#配置文件 /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
138 physical_interface_mappings = public:eth0 #将公共虚拟网络和公共物理网络接口对应起来
[vxlan]
171 enable_vxlan = false #禁止VXLAN覆盖网络
[securitygroup]
156 enable_security_group = true #启用安全组
151 firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver #防火墙驱动
</pre>

### 3.9 配置DHCP代理
<pre>
#配置文件/etc/neutron/dhcp_agent.ini
#在``[DEFAULT]``部分，配置Linuxbridge驱动接口，DHCP驱动并启用隔离元数据，这样在公共网络上的实例就可以通过网络来访问元数据
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver #虚拟接口驱动
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq #开启dhcp驱动
enable_isolated_metadata = true #刷新路由
</pre>

### 3.9.1 配置元数据代理
>Metadata agent>`负责提供配置信息，例如：访问实例的凭证
<pre>
#配置文件/etc/neutron/metadata_agent.ini
#配置元数据主机以及共享密码
nova_metadata_ip = 192.168.56.11 #元数据主机
metadata_proxy_shared_secret = oldboy #共享密码
</pre>

### 3.9.2 为计算节点配置网络服务
<pre>
#/etc/nova/nova.conf 配置文件 nova_api
#在``[neutron]``部分，配置访问参数，启用元数据代理并设置密码
4138[neutron]
url = http://192.168.56.11:9696
auth_url = http://192.168.56.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
------------------------
4155 service_metadata_proxy=true
4158 metadata_proxy_shared_secret = oldboy
</pre>

### 3.9.3 创建软连接
<pre>
#网络服务初始化脚本需要一个超链接 /etc/neutron/plugin.ini``指向ML2插件配置文件/etc/neutron/plugins/ml2/ml2_conf.ini``。如果超链接不存在，使用下面的命令创建它：
# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
</pre>

### 3.9.4 同步数据库
<pre>
# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
-------------------------------
# systemctl restart openstack-nova-api.service #重启nova-api
</pre>

### 3.9.5 启动neutron
<pre>
# systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
# systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
</pre>

### 3.9.6 在keystone上做注册
<pre>
#创建``neutron``服务实体
# openstack service create --name neutron \
>   --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | b6aa961079434531831edfc6ba0b6e63 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
</pre>

### 3.9.7 创建网络服务API端点：
<pre>
# openstack endpoint create --region RegionOne \
>   network public http://192.168.56.11:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | bdd01f1089204f5a8592bcdb7079ebbe |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b6aa961079434531831edfc6ba0b6e63 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.56.11:9696        |
+--------------+----------------------------------+
# openstack endpoint create --region RegionOne   network internal http://192.168.56.11:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5a93224297fc4115887706ed3b913c47 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b6aa961079434531831edfc6ba0b6e63 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.56.11:9696        |
+--------------+----------------------------------+
# openstack endpoint create --region RegionOne   network admin http://192.168.56.11:9696   
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 58fac33743b245aca1fc5620a3b1c253 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b6aa961079434531831edfc6ba0b6e63 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.56.11:9696        |
+--------------+----------------------------------+
openstack service list #查看服务
openstack endpoint list #查看endpoint
</pre>

### 3.9.8 验证neutron是否安装成功  
<pre>
# neutron agent-list
+--------------------------------------+--------------------+-------------------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host                    | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-------------------------+-------------------+-------+----------------+---------------------------+
| 3860a863-9d68-4c40-a072-0dc630f9a79e | Linux bridge agent | linux-node1.example.com |                   | :-)   | True           | neutron-linuxbridge-agent |
| 5eb153c4-a150-4d74-ba00-c511361402f1 | Metadata agent     | linux-node1.example.com |                   | :-)   | True           | neutron-metadata-agent    |
| b92c3229-0835-4b09-a783-c83a355802f2 | DHCP agent         | linux-node1.example.com | nova              | :-)   | True           | neutron-dhcp-agent        |
+--------------------------------------+--------------------+-------------------------+-------------------+-------+----------------+---------------------------+
</pre>