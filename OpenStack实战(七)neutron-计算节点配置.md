# 一 安装neutron计算节点

### 1.1 安装软件包 
<pre>
# yum install openstack-neutron-linuxbridge ebtables ipset
</pre> 

### 2.1 编辑/etc/neutron/neutron.conf 文件并完成如下操作：
<pre>
# 拷贝服务端到计算节点然后修改
# 在``[database]`` 部分，注释所有``connection`` 项，因为计算节点不直接访问数据库。
684 #connection =
# 删除nonva服务
938 [nova]
137 #notify_nova_on_port_status_changes = true
141 #notify_nova_on_port_data_changes = true
30 #core_plugin = ml2
33 service_plugins =
</pre>

### 3.1 配置计算服务来使用网络服务
<pre>
# 编辑``/etc/nova/nova.conf``文件并完成下面的操作：
# 在``[neutron]`` 部分，配置访问参数：
4138 [neutron]
4139 url = http://192.168.56.11:9696
4140 auth_url = http://192.168.56.11:35357
4141 auth_type = password
4142 project_domain_name = default
4143 user_domain_name = default
4144 region_name = RegionOne
4145 project_name = service
4146 username = neutron
4147 password = neutron
</pre>

### 4.1 配置Linuxbridge代理
<pre>
# Linuxbridge代理为实例建立layer－2虚拟网络并且处理安全组规则。
#编辑``/etc/neutron/plugins/ml2/linuxbridge_agent.ini``文件并且完成以下操作：
#在``[linux_bridge]``部分，将公共虚拟网络和公共物理网络接口对应起来：
138 physical_interface_mappings = public:eth0

#在``[vxlan]``部分，禁止VXLAN覆盖网络：
171 enable_vxlan = false

#在 ``[securitygroup]``部分，启用安全组并配置 Linux 桥接 iptables 防火墙驱动：
156 enable_security_group = true
151 firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
</pre>

### 4.1.2 重启计算服务:
<pre>
# systemctl restart openstack-nova-compute.service
</pre>

### 5 启动Linuxbridge代理并配置它开机自启动：

<pre>
# systemctl enable neutron-linuxbridge-agent.service
# systemctl start neutron-linuxbridge-agent.service
</pre>

### 6 验证
<pre>
[root@linux-node1 ~]# source admin-openstack.sh 
[root@linux-node1 ~]# neutron agent-list        
+--------------------------------------+--------------------+-------------------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host                    | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-------------------------+-------------------+-------+----------------+---------------------------+
| 3860a863-9d68-4c40-a072-0dc630f9a79e | Linux bridge agent | linux-node1.example.com |                   | :-)   | True           | neutron-linuxbridge-agent |
| 5eb153c4-a150-4d74-ba00-c511361402f1 | Metadata agent     | linux-node1.example.com |                   | :-)   | True           | neutron-metadata-agent    |
| 70fc7974-7694-49a4-975e-741d238b73eb | Linux bridge agent | linux-node2.example.com |                   | :-)   | True           | neutron-linuxbridge-agent |
| b92c3229-0835-4b09-a783-c83a355802f2 | DHCP agent         | linux-node1.example.com | nova              | :-)   | True           | neutron-dhcp-agent        |
+--------------------------------------+--------------------+-------------------------+-------------------+-------+----------------+---------------------------+
</pre>
---
## 七 创建第一台虚拟机

### 7.1创建提供者网络
![](images/neutrongl.png)     
### 7.2 提供者网络-概述  
![](images/neutronty.png)    
<pre>
 source admin-openstack.sh 
# neutron net-create --shared --provider:physical_network public --provider:network_type flat public-net
>neutron net-create 创建一个网络
>-shared 共享网络
>--provider:physical_network 提供者 public 公共的
>--provider:network_type 提供者的类型  flat 单一扁平网络 
>public-net 网络名称 
#neutron net-list #查看网络
-------------------------
#创建子网
neutron subnet-create --name public-subnet --allocation-pool start=192.168.56.100,end=192.168.56.200 --dns-nameserver 223.5.5.5 --gateway 192.168.56.2 public-net 192.168.56.0/24

>neutron subnet-create 子网创建 
>--name 名称
>--allocation-pool 分配的地址池
>public-net 和网络关联 
# neutron subnet-list  #查看子网
</pre>

## 八 创建m1.nano规格的主机     
      
>默认的最小规格的主机需要512 MB内存。对于环境中计算节点内存不足4 GB的，我们推荐创建只需要64 MB的``m1.nano``规格的主机。若单纯为了测试的目的，请使用``m1.nano``规格的主机来加载CirrOS镜像  
<pre>
# openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
# openstack flavor list  查看规格主机1-5默认0是创建的
</pre>

## 8.2 生成一个秘钥对
>大部分云镜像支持公共密钥认证而不是传统的密码认证。在启动实例前，你必须添加一个公共密钥到计算服务。

### 8.2.1 导入租户``demo``的凭证
<pre>
# source demo-openstack.sh
</pre>
### 8.2.3 生成和添加秘钥对
<pre>
# ssh-keygen -q -N ""
# openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
# openstack keypair list 验证公钥的添加
</pre>

### 8.2.4 增加安全组规则
>默认情况下， ``default``安全组适用于所有实例并且包括拒绝远程访问实例的防火墙规则。对诸如CirrOS这样的Linux镜像，我们推荐至少允许ICMP (ping) 和安全shell(SSH)规则
<pre>
# 允许 ICMP (ping)
# openstack security group rule create --proto icmp default
# 允许安全 shell (SSH) 的访问：
# openstack security group rule create --proto tcp --dst-port 22 default
</pre>
## 九 在公有网络上创建实例
---

### 9.1 确定实例选项
<pre>
# source demo-openstack.sh
# openstack flavor list
+----+-----------+-------+------+-----------+-------+-----------+
| ID | Name      |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+-----------+-------+------+-----------+-------+-----------+
| 0  | m1.nano   |    64 |    1 |         0 |     1 | True      |
| 1  | m1.tiny   |   512 |    1 |         0 |     1 | True      |
| 2  | m1.small  |  2048 |   20 |         0 |     1 | True      |
| 3  | m1.medium |  4096 |   40 |         0 |     2 | True      |
| 4  | m1.large  |  8192 |   80 |         0 |     4 | True      |
| 5  | m1.xlarge | 16384 |  160 |         0 |     8 | True      |
+----+-----------+-------+------+-----------+-------+-----------+
# 一个实例指定了虚拟机资源的大致分配，包括处理器、内存和存储,列出可用类型,这个实例使用``m1.tiny``规格的主机。如果你创建了``m1.nano``这种主机规格，使用``m1.nano``来代替``m1.tiny``
# openstack image list #列出可用镜像
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 0e86ad26-937b-4080-832a-46be90084d2f | cirros | active |
+--------------------------------------+--------+--------+

# openstack security group list 列出可用的安全组
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 3c3df622-8315-4a5e-afec-e908d3074caf | default | Default security group | 6d21b0492bc94412833706d0c19afcee |
+--------------------------------------+---------+------------------------+----------------------------------+

# openstack network list 列出可用网络，下面创建虚拟机需要网络id
+--------------------------------------+------------+--------------------------------------+
| ID                                   | Name       | Subnets                              |
+--------------------------------------+------------+--------------------------------------+
| 1af606e2-5df9-4fd1-981e-ec96c3393595 | public-net | 1b0b6276-70fe-4754-8de9-0feddafb1d5e |
+--------------------------------------+------------+--------------------------------------+
</pre>

### 9.2 创建实例
<pre>
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=1af606e2-5df9-4fd1-981e-ec96c3393595 --security-group default \
  --key-name mykey provider-instance
# 检查实例的状态
# openstack server list
+--------------------------------------+--------+--------+---------------------------+
| ID                                   | Name   | Status | Networks                  |
+--------------------------------------+--------+--------+---------------------------+
| e4200b98-c407-464f-8a43-bf950d249998 | demo-2 | ACTIVE | public-net=192.168.56.103 |
| bc81004b-0649-46c1-bf55-0cadef0bfae5 | demo-1 | ACTIVE | public-net=192.168.56.102 |
+--------------------------------------+--------+--------+---------------------------+
# openstack console url show provider-instance
#验证：
# ping 192.168.56.103
PING 192.168.56.103 (192.168.56.103) 56(84) bytes of data.
64 bytes from 192.168.56.103: icmp_seq=1 ttl=64 time=1.30 ms
64 bytes from 192.168.56.103: icmp_seq=2 ttl=64 time=1.02 ms
64 bytes from 192.168.56.103: icmp_seq=3 ttl=64 time=0.679 ms
64 bytes from 192.168.56.103: icmp_seq=4 ttl=64 time=0.959 ms
64 bytes from 192.168.56.103: icmp_seq=5 ttl=64 time=0.595 ms
64 bytes from 192.168.56.103: icmp_seq=6 ttl=64 time=0.576 ms
--- 192.168.56.103 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 5006ms
rtt min/avg/max/mdev = 0.576/0.856/1.300/0.263 ms

# ssh cirros@192.168.56.103
The authenticity of host '192.168.56.103 (192.168.56.103)' can't be established.
RSA key fingerprint is 50:73:ca:83:d6:18:09:90:5d:65:33:25:34:3f:af:2f.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.56.103' (RSA) to the list of known hosts.
$ 
$ ip ro sh 
default via 192.168.56.2 dev eth0 
169.254.169.254 via 192.168.56.100 dev eth0 
192.168.56.0/24 dev eth0  src 192.168.56.103 
$ ping www.baidu.com
PING www.baidu.com (61.135.169.125): 56 data bytes
64 bytes from 61.135.169.125: seq=0 ttl=128 time=8.722 ms
64 bytes from 61.135.169.125: seq=1 ttl=128 time=13.194 ms
64 bytes from 61.135.169.125: seq=2 ttl=128 time=19.880 ms

--- www.baidu.com ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 8.722/13.932/19.880 ms
$ whoami 
cirros
$ 
# openstack console url show provider-instance 获取url，vnc地址

#排错思路,时间是否同步，selinux是否关闭
# nova service-list 
# neutron agent-list
# nova image-list
清空日志，重新创建虚拟机，tail -f /var/log/nova/* /var/log/neutron/*
grep "ERROR" /var/log/nova/*
grep "ERROR" /var/log/neutron/*
grep "ERROR" /var/log/glance/*
grep "ERROR" /var/log/keystone/*
</pre>
