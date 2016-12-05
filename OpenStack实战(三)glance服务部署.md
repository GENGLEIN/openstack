# 一 glance官方简介
---
OpenStack镜像服务是IaaS的核心服务，如同 :ref:`get_started_conceptual_architecture`所示。它接受磁盘镜像或服务器镜像API请求，和来自终端用户或OpenStack计算组件的元数据定义。它也支持包括OpenStack对象存储在内的多种类型仓库上的磁盘镜像或服务器镜像存储。
大量周期性进程运行于OpenStack镜像服务上以支持缓存。同步复制（Replication）服务保证集群中的一致性和可用性。其它周期性进程包括auditors, updaters, 和 reapers。
OpenStack镜像服务包括以下组件：

glance-api
接收镜像API的调用，诸如镜像发现、恢复、存储。

glance-registry
存储、处理和恢复镜像的元数据，元数据包括项诸如大小和类型。


数据库
存放镜像元数据，用户是可以依据个人喜好选择数据库的，多数的部署使用MySQL或SQLite。

镜像文件的存储仓库
支持多种类型的仓库，它们有普通文件系统、对象存储、RADOS块设备、HTTP、以及亚马逊S3。记住，其中一些仓库仅支持只读方式使用。
元数据定义服务
通用的API，是用于为厂商，管理员，服务，以及用户自定义元数据。这种元数据可用于不同的资源，例如镜像，工件，卷，配额以及集合。一个定义包括了新属性的键，描述，约束以及可以与之关联的资源的类型。

# 二 安装glance先决条件 
---
 安装和配置镜像服务之前，你必须创建创建一个数据库、服务凭证和API端点。
<pre>
用数据库连接客户端以 root 用户连接到数据库服务器：
# mysql -u root -p
# create database glance;
# grant all on glance.* to 'glance'@'localhost' identified by 'glance';
# grant all on glance.* to 'glance'@'%' identified by 'glance';
</pre>
# 三 glance部署安装 
---
### 3.1 安装软件包:
<pre>
yum install openstack-glance
</pre>
### 3.2 编辑文件/etc/glance/glance-api.conf并完成如下：
> 在[database]部分，配置数据库访问
<pre>
641 connection = mysql+pymysql://glance:glance@192.168.56.11/glance
</pre>
### 3.3 编辑文件/etc/glance/glance-registry.conf并完成如下：
<pre>
382 connection = mysql+pymysql://glance:glance@192.168.56.11/glance
</pre>
### 3.4 同步数据库
<pre>
# su -s /bin/sh -c "glance-manage db_sync" glance
# use glance
Database changed
MariaDB [glance]> show tables;
+----------------------------------+
| Tables_in_glance                 |
+----------------------------------+
| artifact_blob_locations          |
| artifact_blobs                   |
| artifact_dependencies            |
| artifact_properties              |
| artifact_tags                    |
| artifacts                        |
| image_locations                  |
| image_members                    |
| image_properties                 |
| image_tags                       |
| images                           |
| metadef_namespace_resource_types |
| metadef_namespaces               |
| metadef_objects                  |
| metadef_properties               |
| metadef_resource_types           |
| metadef_tags                     |
| migrate_version                  |
| task_info                        |
| tasks                            |
+----------------------------------+
</pre>

### 3.5 配置keystone认证

<pre>
# vim /etc/glance/glance-api.conf
1111 [keystone_authtoken]
1112 auth_uri = http://192.168.56.11:5000
1113 auth_url = http://192.168.56.11:35357
1114 memcached_servers = 192.168.56.11:11211
1115 auth_type = password
1116 project_domain_name = default
1117 user_domain_name = default
1118 project_name = service
1119 username = glance
1120 password = glance
---------------------------------------------
1685 [paste_deploy]
1695 flavor = keystone
</pre>

### 3.6 配置keystone认证
<pre>
# vim /etc/glance/glance-registry.conf
 836 [keystone_authtoken]
 837 auth_uri = http://192.168.56.11:5000
 838 auth_url = http://192.168.56.11:35357
 839 memcached_servers = 192.168.56.11:11211
 840 auth_type = password
 841 project_domain_name = default
 842 user_domain_name = default
 843 project_name = service
 844 username = glance
 845 password = glance
-----------------------------------------------
1392 [paste_deploy]
1402 flavor = keystone
</pre>

### 3.7 配置glance镜像存储方式和存储目录
<pre>
# vim /etc/glance/glance-api.conf 
733 [glance_store]
741 stores = file,http   #存储方式
746 default_store = file 
1025 filesystem_store_datadir = /var/lib/glance/images
</pre>

### 3.8 启动镜像服务、配置他们开机启动
<pre>
# systemctl enable openstack-glance-api.service \
  openstack-glance-registry.service

# systemctl start openstack-glance-api.service \
  openstack-glance-registry.service

# netstat -lntup |egrep "9292|9191"
tcp        0      0 0.0.0.0:9292            0.0.0.0:*               LISTEN      9414/python2        
tcp        0      0 0.0.0.0:9191            0.0.0.0:*               LISTEN      9415/python2
</pre>



### 3.9 在keystone上做服务认证
<pre>
# 创建服务
# source  admin-openstack.sh
# openstack service create --name glance \
>   --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | a0b5ee4a3f00452ebebc4e49ca30e485 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
------------------------------------------------------
</pre>

### 3.9.1 创建三个端点服务:
<pre>
# openstack endpoint create --region RegionOne \
>   image public http://192.168.56.11:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 27430dadfa6d4fc6b2e2ace475c56903 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a0b5ee4a3f00452ebebc4e49ca30e485 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.56.11:9292        |
+--------------+----------------------------------+
# openstack endpoint create --region RegionOne   image internal http://192.168.56.11:9292
+-------------------------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ce30c8d5c2a94aed8e1af92d332f5cfd |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a0b5ee4a3f00452ebebc4e49ca30e485 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.56.11:9292        |
+--------------+----------------------------------+
# openstack endpoint create --region RegionOne   image admin http://192.168.56.11:9292   
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e2522dfa8f484d99b0508ad64aeead26 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a0b5ee4a3f00452ebebc4e49ca30e485 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.56.11:9292        |
+--------------+----------------------------------+
# glance image-list  #验证空列表为正常
+----+------+
| ID | Name |
+----+------+
+----+------+
</pre>

### 3.9.2 上传镜像验证操作
<pre>
# source admin-openstack.sh 换窗口需要
# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img #下载镜像
# 上传镜像：
openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2016-10-26T08:47:02Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/0e86ad26-937b-4080-832a-46be90084d2f/file |
| id               | 0e86ad26-937b-4080-832a-46be90084d2f                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 6044635670c4476189d549a2db09e175                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2016-10-26T08:47:03Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
</pre>

### 3.9.3 查看镜像
<pre>
# glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| 0e86ad26-937b-4080-832a-46be90084d2f | cirros |
+--------------------------------------+--------+
# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 0e86ad26-937b-4080-832a-46be90084d2f | cirros | active |
+--------------------------------------+--------+--------+
# cd /var/lib/glance/images/
# ll -l
-rw-r----- 1 glance glance 13287936 10月 26 16:47 0e86ad26-937b-4080-832a-46be90084d2f
</pre>