
# 一 keystone验证服务官方简介
---
OpenStack:term:`Identity service`为认证管理，授权管理和服务目录服务管理提供单点整合。其它OpenStack服务将身份认证服务当做通用统一API来使用。此外，提供用户信息但是不在OpenStack项目中的服务（如LDAP服务）可被整合进先前存在的基础设施中,为了从identity服务中获益，其他的OpenStack服务需要与它合作。当某个OpenStack服务收到来自用户的请求时，该服务询问Identity服务，验证该用户是否有权限进行此次请求    
身份服务包含这些组件：

服务器
一个中心化的服务器使用RESTful 接口来提供认证和授权服务。

驱动
驱动或服务后端被整合进集中式服务器中。它们被用来访问OpenStack外部仓库的身份信息, 并且它们可能已经存在于OpenStack被部署在的基础设施（例如，SQL数据库或LDAP服务器）中。

模块
中间件模块运行于使用身份认证服务的OpenStack组件的地址空间中。这些模块拦截服务请求，取出用户凭据，并将它们送入中央是服务器寻求授权。中间件模块和OpenStack组件间的整合使用Python Web服务器网关接口。

当安装OpenStack身份服务，用户必须将之注册到其OpenStack安装环境的每个服务。身份服务才可以追踪那些OpenStack服务已经安装，以及在网络中定位它们。

yum install -y net-tools vim wget lrzsz tree screen lsof tcpdump



# 二 keystone安装部署
---
### 2.1先决条件  
* 2.1.1在你配置 OpenStack 身份认证,用户管理服务前，你必须创建一个数据库和管理员令牌
<pre>
# MariaDB [(none)]> create database keystone;
Query OK, 1 row affected (0.00 sec)

# MariaDB [(none)]> grant all on keystone.* to 'keystone'@'localhost' identified by 'keystone';
Query OK, 0 rows affected (0.00 sec)

# MariaDB [(none)]> grant all on keystone.* to 'keystone'@'%' identified by 'keystone';         
Query OK, 0 rows affected (0.00 sec)
# openssl rand -hex 10
</pre>

### 2.2安装keyston
<pre>
# yum install -y openstack-keystone httpd mod_wsgi memcached python-memcached
httpd #keyston需要Apache来运行
mod_wsgi #python接口
memcached #做缓存
</pre>

### 2.3 编辑文件 /etc/keystone/keystone.conf 并完成如下动作
* 1 在``[DEFAULT]``部分，定义初始管理令牌的值
<pre>
admin_token = 354018dcf3ab775f6316
openssl rand -hex 10
</pre>

* 2 在 [database] 部分，配置数据库访问
<pre>
549 connection = mysql+pymysql://keystone:keystone@192.168.56.11/keystone
</pre>

* 3 在``[token]``部分，配置Fernet UUID令牌的提供者
<pre>
2005 provider = fernet
</pre>

* 4 在``[memcache] 部分，配置memcache提供地址
<pre>
1252 servers = 192.168.56.11:11211 
2010 driver = memcache  token存储位置默认是数据库
</pre>

### 2.4 初始化身份认证服务的数据库
<pre>
# su -s /bin/sh -c "keystone-manage db_sync" keystone
#  cd /var/log/keystone/ #用什么用户同步属主就属于谁,如果用root同步记得改这里
-rw-r--r-- 1 keystone keystone 4402 Oct 25 22:58 keystone.log
# use keystone #查看表
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]> show tables;
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| domain                 |
| endpoint               |
| endpoint_group         |
| federated_user         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids       

</pre>

### 2.5 初始化Fernet keys
<pre>
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# cd /etc/keystone/
# ll -l
one  2303 Sep 22 20:06 default_catalog.templates
one    22 Oct 25 23:13 fernet-keys #生成证书目录
one  2400 Sep 22 20:06 keystone-paste.ini
one 73171 Oct 25 22:51 keystone.conf
one  1046 Sep 22 20:06 logging.conf
one  9699 Sep 22 20:06 policy.json
one   665 Sep 22 20:06 sso_callback_template.html
</pre>

### 2.6 启动memcache
<pre>
# systemctl enable memcached
# systemctl start memcached 
# lsof -i :11211
COMMAND     PID      USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
memcached 19475 memcached   26u  IPv4  69044      0t0  TCP *:memcache (LISTEN)
memcached 19475 memcached   27u  IPv6  69045      0t0  TCP *:memcache (LISTEN)
memcached 19475 memcached   28u  IPv4  69048      0t0  UDP *:memcache 
memcached 19475 memcached   29u  IPv4  69048      0t0  UDP *:memcache 
memcached 19475 memcached   30u  IPv4  69048      0t0  UDP *:memcache 
memcached 19475 memcached   31u  IPv4  69048      0t0  UDP *:memcache 
memcached 19475 memcached   32u  IPv6  69049      0t0  UDP *:memcache 
memcached 19475 memcached   33u  IPv6  69049      0t0  UDP *:memcache 
memcached 19475 memcached   34u  IPv6  69049      0t0  UDP *:memcache 
memcached 19475 memcached   35u  IPv6  69049      0t0  UDP *:memcache 
# cat /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS=""
</pre>

### 2.7 编辑``/etc/httpd/conf/httpd.conf`` 文件，配置``ServerName`` 选项为控制节点，来启动keystone	
<pre>
95 ServerName 192.168.56.11:80
</pre> 

### 2.8 用下面的内容创建文件 /etc/httpd/conf.d/wsgi-keystone.conf
<pre>
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>
</pre>

### 2.9 启动 Apache HTTP 服务并配置其随系统启动
<pre>
# systemctl enable httpd.service
# systemctl start httpd.service
# 查看端口号，如有问题打开debug
# lsof -i :5000
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   19668   root    6u  IPv6  70211      0t0  TCP *:commplex-main (LISTEN)
httpd   19679 apache    6u  IPv6  70211      0t0  TCP *:commplex-main (LISTEN)
httpd   19680 apache    6u  IPv6  70211      0t0  TCP *:commplex-main (LISTEN)
httpd   19690 apache    6u  IPv6  70211      0t0  TCP *:commplex-main (LISTEN)
httpd   19691 apache    6u  IPv6  70211      0t0  TCP *:commplex-main (LISTEN)
httpd   19692 apache    6u  IPv6  70211      0t0  TCP *:commplex-main (LISTEN)
# lsof -i :35357
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   19668   root    8u  IPv6  70215      0t0  TCP *:openstack-id (LISTEN)
httpd   19679 apache    8u  IPv6  70215      0t0  TCP *:openstack-id (LISTEN)
httpd   19680 apache    8u  IPv6  70215      0t0  TCP *:openstack-id (LISTEN)
httpd   19690 apache    8u  IPv6  70215      0t0  TCP *:openstack-id (LISTEN)
httpd   19691 apache    8u  IPv6  70215      0t0  TCP *:openstack-id (LISTEN)
httpd   19692 apache    8u  IPv6  70215      0t0  TCP *:openstack-id (LISTEN)
# ps -ef |grep keystone
keystone  19669  19668  0 23:29 ?        00:00:00 (wsgi:keystone- -DFOREGROUND
keystone  19670  19668  0 23:29 ?        00:00:00 (wsgi:keystone- -DFOREGROUND
keystone  19671  19668  0 23:29 ?        00:00:00 (wsgi:keystone- -DFOREGROUND
keystone  19672  19668  0 23:29 ?        00:00:00 (wsgi:keystone- -DFOREGROUND
keystone  19673  19668  0 23:29 ?        00:00:00 (wsgi:keystone- -DFOREGROUND
</pre>

### 2.9.1 设置环境变量连接keystone来创建域、项目、用户和角色
<pre>
配置认证令牌	
# export OS_TOKEN=354018dcf3ab775f6316
配置端点URL
# export OS_URL=http://192.168.56.11:35357/v3
配置认证 API 版本
# export OS_IDENTITY_API_VERSION=3
</pre>

### 2.9.2 创建default域
<pre>
# openstack domain create --description "Default Domain" default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Default Domain                   |
| enabled     | True                             |
| id          | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| name        | default                          |
+-------------+----------------------------------+
</pre>

### 2.9.3 创建admin项目
<pre>
# openstack project create --domain default \
>   --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| domain_id   | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled     | True                             |
| id          | 6044635670c4476189d549a2db09e175 |
| is_domain   | False                            |
| name        | admin                            |
| parent_id   | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
+-------------+----------------------------------+

</pre>

### 2.9.4 创建admin用户
<pre>
# openstack user create --domain default \
>   --password-prompt admin
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled   | True                             |
| id        | 185b134482284ab196ae000ea65f894e |
| name      | admin                            |
+-----------+----------------------------------+
密码：admin

</pre>

### 2.9.5 创建admin角色
<pre>
# openstack role create admin
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 6933bef1c4e649ab9fd0348af98c5c20 |
| name      | admin                            |
+-----------+----------------------------------+
</pre>

### 2.9.6 把admin项目添加到admin用户上，并且授权admin角色
<pre>
# openstack role add --project admin --user admin admin
</pre>

### 2.9.7 创建demo项目
<pre>
# openstack project create --domain default \
>   --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled     | True                             |
| id          | 6d21b0492bc94412833706d0c19afcee |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
+-------------+----------------------------------+


</pre>

### 2.9.8 创建demo用户
<pre>
 openstack user create --domain default \
>   --password-prompt demo
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled   | True                             |
| id        | 6eb30962b13b42ce948cf59caf9f7cd5 |
| name      | demo                             |
+-----------+----------------------------------+
密码：demo

</pre>
### 2.9.9 创建user角色
<pre>
# openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | ac819934294149ad855596e47734f67d |
| name      | user                             |
+-----------+----------------------------------+
</pre>

### 2.9.9.1 把demo项目添加到demo用户，并且授权user角色

<pre>
# openstack role add --project demo --user demo user
</pre>

### 2.9.9.2 创建service项目
>创建service项目，创建glance,nova,neutron用户，加入service项目，授权admin角色，创建这些用户就是为了给后面这些服务做认证glance,nova,neutron
<pre>
# openstack project create --domain default \
>   --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled     | True                             |
| id          | 27af3021bcc04b3596768f2aa710c628 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
+-------------+----------------------------------+
-----------------------------------------------------
#创建glance用户
# openstack user create --domain default --password-prompt glance   
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled   | True                             |
| id        | 40996f0897514733912368e9915752fe |
| name      | glance                           |
+-----------+----------------------------------+
密码:glance
#把glance用户加入service项目，并且授权admin角色
# openstack role add --project service --user glance admin
------------------------------------------------------------------
# 创建nova用户
# openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled   | True                             |
| id        | c75209c6572846e79bbceafa09b76ff7 |
| name      | nova                             |
+-----------+----------------------------------+
密码:nova
#把nova用户加入service项目，并且授权admin角色
# openstack role add --project service --user nova admin
-----------------------------------------------------------------
# 创建neutron用户
# openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | ca2b66fc49b34c8b84ddc5cb1b6f0e00 |
| enabled   | True                             |
| id        | eb8555b078a7410c9ff0a8f682465f7e |
| name      | neutron                          |
+-----------+----------------------------------+
密码：neutron
# 把neutron用户加入service项目，并且授权admin角色
# openstack role add --project service --user neutron admin
</pre>

# 三 服务注册
---
### 3.1 创建service服务
<pre>
# openstack service create \
>   --name keystone --description "OpenStack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 8b5ab5763fa142c4b57417e77a08a0d8 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
</pre>

### 3.2 创建public公共的

<pre>
# openstack endpoint create --region RegionOne \
>   identity public http://192.168.56.11:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a4cefe15fd7c4e378f129b084290237b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8b5ab5763fa142c4b57417e77a08a0d8 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.56.11:5000/v3     |
+--------------+----------------------------------+
</pre>

### 3.3 创建internal私有的
<pre>
# openstack endpoint create --region RegionOne   identity internal http://192.168.56.11:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8e517def446040bc9a2eda9781d103e9 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8b5ab5763fa142c4b57417e77a08a0d8 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.56.11:5000/v3     |
+--------------+----------------------------------+
</pre>

### 3.4 创建admin
<pre>
# openstack endpoint create --region RegionOne   identity admin http://192.168.56.11:35357/v3  
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1202b779a6774692b9478bbd9068f633 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8b5ab5763fa142c4b57417e77a08a0d8 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.56.11:35357/v3    |
+--------------+----------------------------------+
# openstack endpoint list
# openstack service list

</pre>

> 用户认证 
User:用户、Tenant:租户项目、Token:令牌、Role:角色

> 服务目录 Service:服务、Endpoint:端口
3个端点，访问点   
        public 全局访问，
		private 局域网服务之间访问
		admin 管理员
------------------------------------------------------------

# 四 验证keystone安装
### 4.1 因为安全性的原因，关闭临时认证令牌机制
<pre>
# unset OS_TOKEN OS_URL
</pre>

### 4.2 作为 admin 用户，请求认证令牌
<pre>
# openstack --os-auth-url http://192.168.56.11:35357/v3 \
>   --os-project-domain-name default --os-user-domain-name default \
>   --os-project-name admin --os-username admin token issue
Password: 
+------------+--------------------------------------------------------------------------------------------+
| Field      | Value                                                                                      |
+------------+--------------------------------------------------------------------------------------------+
| expires    | 2016-10-26T04:13:15.472149Z                                                                |
| id         | gAAAAABYEB9LvDFy_8tk-oXHRaFanyCPFEC-2yBBKLBurD1xWNDyIfBDFO9WYVFelXgl-                      |
|            | VLT9bnkVhMrnhKsr1x8mxA6dU3TEpuxhJ0YVIIlkUVEamWlT6THuYj3l2lG-                               |
|            | 5O84lNSbGbw_tPXZGenwe2hmB7lRa9H8q2w_Z-ZreoV0cRPExNxfA4                                     |
| project_id | 6044635670c4476189d549a2db09e175                                                           |
| user_id    | 185b134482284ab196ae000ea65f894e                                                           |
+------------+--------------------------------------------------------------------------------------------+

</pre>

### 4.3 作为``demo`` 用户，请求认证令牌
<pre>
# openstack --os-auth-url http://192.168.56.11:5000/v3 \
>   --os-project-domain-name default --os-user-domain-name default \
>   --os-project-name demo --os-username demo token issue
Password: 
+------------+--------------------------------------------------------------------------------------------+
| Field      | Value                                                                                      |
+------------+--------------------------------------------------------------------------------------------+
| expires    | 2016-10-26T04:15:58.708568Z                                                                |
| id         | gAAAAABYEB_vQaeSTs81x0Hb7j8SM6rJwoDpbdZfHWXA8JnlYTnCGVn9bKXwfmhlh7Qa3m1CelB07UguZbsAflUPtb |
|            | sW5XIjVxa2n_R-iWzg1ULWVvSymYXfWUknqMrtwsXmXYtztB1pR0JSNwMCVyoS-                            |
|            | 6q8qFvA3U_75jxOZR6fMbk7ooqAAsY                                                             |
| project_id | 6d21b0492bc94412833706d0c19afcee                                                           |
| user_id    | 6eb30962b13b42ce948cf59caf9f7cd5                                                           |
+------------+--------------------------------------------------------------------------------------------+
</pre>

# 五 创建openstack客户端环境脚本
* 5.1 编辑admin-openstack.sh文件并添加如下内容
<pre>
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.56.11:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
</pre>

* 5.2 编辑demo-openstack.sh并添加如下内容
<pre> 
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.56.11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
</pre>

* 5.3 加载环境变量，请求认证令牌
<pre>
# source  admin-openstack.sh
# openstack token issue
+------------+--------------------------------------------------------------------------------------------+
| Field      | Value                                                                                      |
+------------+--------------------------------------------------------------------------------------------+
| expires    | 2016-10-26T08:05:02.598446Z                                                                |
| id         | gAAAAABYEFWeOrZIvT-_z2OkGCM4c8-lTeXGct4_IM4r_xmEra0mkoqPX_Cujez2UzrUkwMEsC8TQqzrm3r44Hf4Tk |
|            | WWTwVnoSJ28J5kKDNIgG4AvH2U3BrAXVOe3qkUmVDL4jQRl4u-mvNBjDuKDYsX6UkKUi--                     |
|            | ZUoF4VDzCmzWqhPwvn0QdFc                                                                    |
| project_id | 6d21b0492bc94412833706d0c19afcee                                                           |
| user_id    | 6eb30962b13b42ce948cf59caf9f7cd5                                                           |
+------------+--------------------------------------------------------------------------------------------+

</pre>

10 干 牛 拉 















