# 一 官方简介
--- 
---
###使用OpenStack计算服务来托管和管理云计算系统。OpenStack计算服务是基础设施即服务(IaaS)系统的主要部分，模块主要由Python实现。OpenStack计算组件请求OpenStack Identity服务进行认证；请求OpenStack Image服务提供磁盘镜像；为OpenStack dashboard提供用户与管理员接口。磁盘镜像访问限制在项目与用户上；配额以每个项目进行设定（例如，每个项目下可以创建多少实例）。OpenStack组件可以在标准硬件上水平大规模扩展，并且下载磁盘镜像启动虚拟机实例。

#### OpenStack计算服务由下列组件所构成：

####nova-api 服务
####接收和响应来自最终用户的计算API请求。此服务支持OpenStack计算服务API，Amazon EC2 API，以及特殊的管理API用于赋予用户做一些管理的操作。它会强制实施一些规则，发起多数的编排活动，例如运行一个实例。

####nova-api-metadata 服务
####接受来自虚拟机发送的元数据请求。``nova-api-metadata``服务一般在安装``nova-network``服务的多主机模式下使用。更详细的信息，请参考OpenStack管理员手册中的链接`Metadata service <http://docs.openstack.org/admin-guide/compute-networking-nova.html#metadata-service>`__ in the OpenStack Administrator Guide。


# 二 配置安装
---
### 2.1 安装软件包
<pre>
# 同步计算节点和控制节点时间
# yum install openstack-nova-compute
</pre>

### 2.2 确定您的计算节点是否支虚拟化
<pre>
egrep -c '(vmx|svm)' /proc/cpuinfo
1
</pre>

### 2.3 配置nova计算节点，因为和控制节点差别不是很大，我们这里直接拷贝修改
[官方地址](http://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/nova-compute-install.html)
<pre>
# scp  /etc/nova/nova.conf 192.168.56.12:/etc/nova
# 查看权限
-rw-r----- 1 root nova 184330 10月 27 14:52 nova.conf
#修改配置文件，删除数据库配置，注释
3128 connection=mysql+pymysql://nova:nova@192.168.56.11/nova
3128 #connection= #修改后
2168 connection=mysql+pymysql://nova:nova@192.168.56.11/nova_api
2168 #connection=
# 修改vncproxy
5532 novncproxy_base_url=http://192.168.56.11:6080/vnc_auto.html
# 监听地址，修改本地
5427 vncserver_listen=0.0.0.0
# proxyclient修改为本地
5451 vncserver_proxyclient_address=192.168.56.12
5385 enabled=true  #改为true
3683 virt_type=kvm #打开kvm
</pre>
### 2.4 启动nova计算节点-nova-compute
<pre>
# systemctl enable libvirtd.service openstack-nova-compute.service
# systemctl start libvirtd.service openstack-nova-compute.service
# 查看服务是否启动
# ps -ef |grep nova-compute
nova       3965      1  5 15:10 ?        00:00:05 /usr/bin/python2 /usr/bin/nova-compute
root       4021   2585  0 15:11 pts/0    00:00:00 grep --color=auto nova-compute
</pre>

### 2.5 在控制节点查看计算节点是否正常
<pre>
# source  admin-openstack.sh
# openstack host list
+-------------------------+-------------+----------+
| Host Name               | Service     | Zone     |
+-------------------------+-------------+----------+
| linux-node1.example.com | consoleauth | internal |
| linux-node1.example.com | conductor   | internal |
| linux-node1.example.com | scheduler   | internal |
| linux-node2.example.com | compute     | nova     |
+-------------------------+-------------+----------+
# 出现 linux-node2.example.com，compute，nova 说明计算节点已经启动

</pre>
### 2.6 查看nova服务控制节点
<pre>
# nova service-list
+----+------------------+-------------------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host                    | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-------------------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | linux-node1.example.com | internal | enabled | up    | 2016-10-27T07:40:08.000000 | -               |
| 2  | nova-conductor   | linux-node1.example.com | internal | enabled | up    | 2016-10-27T07:40:08.000000 | -               |
| 3  | nova-scheduler   | linux-node1.example.com | internal | enabled | up    | 2016-10-27T07:40:08.000000 | -               |
| 6  | nova-compute     | linux-node2.example.com | nova     | enabled | up    | 2016-10-27T07:40:13.000000 | -               |
+----+------------------+-------------------------+----------+---------+-------+----------------------------+----------
</pre>
### 2.7 查看nova和glance连接是否正常
<pre>
# nova image-list
+--------------------------------------+--------+--------+--------+
| ID                                   | Name   | Status | Server |
+--------------------------------------+--------+--------+--------+
| 0e86ad26-937b-4080-832a-46be90084d2f | cirros | ACTIVE |        |
+--------------------------------------+--------+--------+--------+
</pre>
