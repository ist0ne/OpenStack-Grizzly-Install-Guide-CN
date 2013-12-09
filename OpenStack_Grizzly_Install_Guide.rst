==========================================================
  OpenStack Grizzly 安装指南
==========================================================

:Version: 1.0
:Source: https://github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN
:Keywords: 单节点OpenStack安装, Grizzly, Quantum, Nova, Keystone, Glance, Horizon, Cinder, LinuxBridge, KVM, Ubuntu Server 12.04 (64 bits).

作者
==========

`Shi Dongliang <http://stone.so>`_ <istone2008@gmail.com>

本指南fork自
`Bilel Msekni <https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide>`_ 
的git仓库。向第一作者致敬！

内容列表
=================

::

  0. 简介
  1. 测试环境
  2. 准备宿主机
  3. 设置Keystone
  4. 设置Glance
  5. 设置Quantum
  6. 设置Nova
  7. 设置Cinder
  8. 设置Horizon
  9. 你的第一个VM

0. 简介
==============

OpenStack Grizzly安装指南旨在让你轻松创建自己的OpenStack云平台。

状态: Stable


1. 测试环境
====================

:节点角色: NICs
:单节点: eth0 (10.10.100.51), eth1 (192.168.100.51)

**注意1:** 多节点部署键OVS_MultiNode分支

**注意2:** 你总是可以使用dpkg -s <packagename>确认你使用的是grizzly软件包(版本: 2013.1)

**注意3:** 这个是当前网络架构

.. image:: http://i.imgur.com/JyMokiY.jpg

2. 准备节点
===============

2.1. 准备Ubuntu
-----------------

* 安装好Ubuntu 12.04 Server 64bits后, 进入sudo模式直到完成本指南::

   sudo su -

* 添加Grizzly仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main >> /etc/apt/sources.list.d/grizzly.list

* 升级系统::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

2.2.设置网络
------------

* 如下编辑网卡配置文件/etc/network/interfaces:: 

   #Not internet connected(used for OpenStack management)
   auto eth0
   iface eth0 inet static
   address 10.10.100.51
   netmask 255.255.255.0

   #For Exposing OpenStack API over the internet
   auto eth1
   iface eth1 inet static
   address 192.168.100.51
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8


* 重启网络服务::

   service networking restart

2.3. 安装MySQL和RabbitMQ
------------

* 安装MySQL并为root用户设置密码::

   apt-get install -y mysql-server python-mysqldb

* 配置mysql监听所有网络接口请求::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* 安装RabbitMQ::

   apt-get install -y rabbitmq-server 

* 安装NTP服务::

   apt-get install -y ntp


3. 配置Keystone
=============

* 安装keystone软件包::

   apt-get install -y keystone

* 确认keystone在运行::

   service keystone status

* 为keystone创建MySQL数据库::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;

* 在/etc/keystone/keystone.conf中设置连接到新创建的数据库::

   connection = mysql://keystoneUser:keystonePass@10.10.100.51/keystone

* 重启身份认证服务并同步数据库::

   service keystone restart
   keystone-manage db_sync

* 使用git仓库中脚本填充keystone数据库： `脚本文件夹 <https://github.com/ist0ne/OpenStack-Grizzly-Install-Guide/tree/master/KeystoneScripts>`_ ::

   #注意在执行脚本前请按你的网卡配置修改HOST_IP和HOST_IP_EXT

   wget https://raw.github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN/master/KeystoneScripts/keystone_basic.sh
   wget https://raw.github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN/master/KeystoneScripts/keystone_endpoints_basic.sh

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* 创建一个简单的凭据文件，这样稍后就不会因为输入过多的环境变量而感到厌烦::

   vi creds-admin

   #Paste the following:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://192.168.100.51:5000/v2.0/"

   # Load it:
   source creds-admin

* 通过命令行列出Keystone中添加的用户::

   keystone user-list

4. 设置Glance
=============

* 安装Glance::

   apt-get install -y glance

* 确保glance服务在运行::

   service glance-api status
   service glance-registry status

* 为Glance创建MySQL数据库::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   quit;

* 按下面更新/etc/glance/glance-api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   delay_auth_decision = true
   auth_host = 10.10.100.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 按下面更新/etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.100.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 按下面更新/etc/glance/glance-api.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.100.51/glance

* 和::

   [paste_deploy]
   flavor = keystone
   
* 按下面更新/etc/glance/glance-registry.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.100.51/glance

* 和::

   [paste_deploy]
   flavor = keystone

* 重启glance-api和glance-registry服务::

   service glance-api restart; service glance-registry restart

* 同步glance数据库::

   glance-manage db_sync

* 重启服务使配置生效::

   service glance-registry restart; service glance-api restart

* 测试Glance, 从网络上传cirros云镜像::

   glance image-create --name cirros --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

   注意：通过此镜像创建的虚拟机可通过用户名/密码登陆， 用户名：cirros 密码：cubswin:)

* 本地创建Ubuntu云镜像::

   wget http://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img
   glance add name="Ubuntu 12.04 cloudimg amd64" is_public=true container_format=ovf disk_format=qcow2 < ./precise-server-cloudimg-amd64-disk1.img

* 列出镜像检查是否上传成功::

   glance image-list

5. 设置Quantum
=============

5.2. Quantum-*
------------

* 安装Quantum组件::

   apt-get install -y quantum-server quantum-plugin-linuxbridge quantum-plugin-linuxbridge-agent dnsmasq quantum-dhcp-agent quantum-l3-agent 


* 创建数据库::

   mysql -u root -p
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';
   quit; 

* 确认Quantum组件在运行::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i status; done
   
* 编辑/etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.100.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑OVS配置文件/etc/quantum/plugins/linuxbridge/linuxbridge_conf.ini:: 

   # under [DATABASE] section  
   sql_connection = mysql://quantumUser:quantumPass@10.10.100.51/quantum
   # under [LINUX_BRIDGE] section
   physical_interface_mappings = physnet1:eth1
   # under [VLANS] section
   tenant_network_type = vlan
   network_vlan_ranges = physnet1:1000:2999

* 更新/etc/quantum/metadata_agent.ini::

   # The Quantum user information for accessing the Quantum API.
   auth_url = http://10.10.100.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

   # IP address used by Nova metadata server
   nova_metadata_ip = 10.10.100.51

   # TCP Port used by Nova metadata server
   nova_metadata_port = 8775

   metadata_proxy_shared_secret = helloOpenStack

* 编辑/etc/quantum/quantum.conf::

   core_plugin = quantum.plugins.linuxbridge.lb_quantum_plugin.LinuxBridgePluginV2

   [keystone_authtoken]
   auth_host = 10.10.100.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

* 编辑/etc/quantum/l3_agent.ini::

   [DEFAULT]
   interface_driver = quantum.agent.linux.interface.BridgeInterfaceDriver

* 编辑/etc/quantum/dhcp_agent.ini::

   [DEFAULT]
   interface_driver = quantum.agent.linux.interface.BridgeInterfaceDriver
   dhcp_driver = quantum.agent.linux.dhcp.Dnsmasq
   use_namespaces = True
   signing_dir = /var/cache/quantum
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   auth_url = http://10.10.100.51:35357/v2.0
   dhcp_agent_manager = quantum.agent.dhcp_agent.DhcpAgentWithStateReport
   root_helper = sudo quantum-rootwrap /etc/quantum/rootwrap.conf
   state_path = /var/lib/quantum

* 重启quantum所有服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done
   service dnsmasq restart

*注意: 如果有服务运行在53端口，'dnsmasq'重启失败。 你可以kill掉那个服务器后再重启'dnsmasq'

6. 设置Nova
===========

6.1 KVM
------------------

* 确保你的硬件启用virtualization::

   apt-get install cpu-checker
   kvm-ok

* 现在安装kvm并配置它::

   apt-get install -y kvm libvirt-bin pm-utils

* 在/etc/libvirt/qemu.conf配置文件中启用cgroup_device_acl数组::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* 删除默认的虚拟网桥 ::

   virsh net-destroy default
   virsh net-undefine default

* 更新/etc/libvirt/libvirtd.conf配置文件::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* E编辑libvirtd_opts变量在/etc/init/libvirt-bin.conf配置文件中::

   env libvirtd_opts="-d -l"

* 编辑/etc/default/libvirt-bin文件 ::

   libvirtd_opts="-d -l"

* 重启libvirt服务使配置生效::

   service libvirt-bin restart

6.2 Nova-*
------------------

* 安装nova组件::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor nova-compute-kvm

   注意：如果你的宿主机不支持kvm虚拟化，可把nova-compute-kvm换成nova-compute-qemu
   同时/etc/nova/nova-compute.conf配置文件中的libvirt_type=qemu

* 检查nova服务是否正常启动::

   cd /etc/init.d/; for i in $( ls nova-* ); do service $i status; cd; done

* 为Nova创建Mysql数据库::

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.100.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* 如下修改/etc/nova/nova.conf::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.100.51
   nova_url=http://10.10.100.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.100.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.100.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.100.51
   vncserver_listen=0.0.0.0
   
   # Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack
   
   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://10.10.100.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://10.10.100.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.QuantumLinuxBridgeVIFDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver
  
   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* 修改/etc/nova/nova-compute.conf::

   [DEFAULT]
   libvirt_type=kvm
   compute_driver=libvirt.LibvirtDriver
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.QuantumLinuxBridgeVIFDriver
    
* 同步数据库::

   nova-manage db sync

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list

7. 设置Cinder
===========

* 安装软件包::

   apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* 配置iscsi服务::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* 重启服务::
   
   service iscsitarget start
   service open-iscsi start

* 为Cinder创建Mysql数据库::

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
   quit;

* 如下配置/etc/cinder/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 192.168.100.51
   service_port = 5000
   auth_host = 10.10.100.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* 编辑/etc/cinder/cinder.conf::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.10.100.51/cinder
   api_paste_config = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* 接下来同步数据库::

   cinder-manage db sync

* 最后别忘了创建一个卷组命名为cinder-volumes::

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* 创建物理卷和卷组::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

**注意:** 重启后卷组不会自动挂载 (点击`这个 <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_ 设置在重启后自动挂载) 
重启cinder服务::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done

* 确认cinder服务在运行::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done

8. 设置Horizon
===========

* 如下安装horizon ::

   apt-get install openstack-dashboard memcached

* 如果你不喜欢OpenStack ubuntu主题, 你可以停用它::

   dpkg --purge openstack-dashboard-ubuntu-theme

* 重启Apache和memcached服务::

   service apache2 restart; service memcached restart

现在你可以访问OpenStack **192.168.100.51/horizon** ，使用 **admin:admin_pass** 认证.

9. 创建虚拟机
================

网络拓扑如下：

.. image:: http://i.imgur.com/WdRDVZJ.png

9.1. 为admin租户创建内网、外网、路由器和虚拟机
------------------

* 设置环境变量::

   # cat creds-admin

   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://192.168.100.51:5000/v2.0/"

* 使环境变量生效::

   # source creds-admin

* 列出已创建的用户::

   # keystone user-list

   +----------------------------------+---------+---------+------------------+
   |                id                |   name  | enabled |      email       |
   +----------------------------------+---------+---------+------------------+
   | c815f963fef54f37b0ac84a6a7eca8b4 |  admin  |   True  |  admin@leju.com  |
   | f30d6d67936e41869117b42e5403255c |  cinder |   True  | cinder@leju.com  |
   | 5ec7e55586004aabb6a9ecc8247ba751 |  glance |   True  | glance@leju.com  |
   | 197c373a254749f2b5cec7c91ef14c88 |   nova  |   True  |  nova@leju.com   |
   | 8fec2c89a87d43f19c9e7d487001efa3 | quantum |   True  | quantum@leju.com |
   +----------------------------------+---------+---------+------------------+

* 列出已创建的租户::

   # keystone tenant-list

   +----------------------------------+---------+---------+
   |                id                |   name  | enabled |
   +----------------------------------+---------+---------+
   | 8c0104041b034df3a79c17a9517dd3f9 |  admin  |   True  |
   | 2b376839187441c5888d35411e8ff8b0 | service |   True  |
   +----------------------------------+---------+---------+

* 为admin租户创建网络::

   # quantum net-create --tenant-id 8c0104041b034df3a79c17a9517dd3f9 net_admin

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | fed2d721-41d1-428f-b0a3-41ac8f7a51a1 |
   | name                      | net_admin                            |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 1                                    |
   | router:external           | False                                |
   | shared                    | False                                |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | 8c0104041b034df3a79c17a9517dd3f9     |
   +---------------------------+--------------------------------------+

# 为admin租户创建子网::

   # quantum subnet-create --tenant-id 8c0104041b034df3a79c17a9517dd3f9 net_admin 172.16.100.0/24

   Created a new subnet:
   +------------------+----------------------------------------------------+
   | Field            | Value                                              |
   +------------------+----------------------------------------------------+
   | allocation_pools | {"start": "172.16.100.2", "end": "172.16.100.254"} |
   | cidr             | 172.16.100.0/24                                    |
   | dns_nameservers  |                                                    |
   | enable_dhcp      | True                                               |
   | gateway_ip       | 172.16.100.1                                       |
   | host_routes      |                                                    |
   | id               | fb141492-8aa1-437b-8192-315e19e7f4d2               |
   | ip_version       | 4                                                  |
   | name             |                                                    |
   | network_id       | fed2d721-41d1-428f-b0a3-41ac8f7a51a1               |
   | tenant_id        | 8c0104041b034df3a79c17a9517dd3f9                   |
   +------------------+----------------------------------------------------+

* 为admin租户创建路由器::

   # quantum router-create --tenant-id 8c0104041b034df3a79c17a9517dd3f9 router_admin

   Created a new router:
   +-----------------------+--------------------------------------+
   | Field                 | Value                                |
   +-----------------------+--------------------------------------+
   | admin_state_up        | True                                 |
   | external_gateway_info |                                      |
   | id                    | 76d8ac10-a6df-4dfa-b691-297da374c811 |
   | name                  | router_admin                         |
   | status                | ACTIVE                               |
   | tenant_id             | 8c0104041b034df3a79c17a9517dd3f9     |
   +-----------------------+--------------------------------------+

* 列出路由代理类型::

   # quantum agent-list

   +--------------------------------------+--------------------+-----------+-------+----------------+
   | id                                   | agent_type         | host      | alive | admin_state_up |
   +--------------------------------------+--------------------+-----------+-------+----------------+
   | 2b68d118-c4bb-44a0-8387-678c5bdb1653 | L3 agent           | openstack | :-)   | True           |
   | 7b42460c-cffd-494f-94b1-c6b4f3b5e102 | DHCP agent         | openstack | :-)   | True           |
   | e443fbf2-398c-47ab-89d9-5d9907217379 | Open vSwitch agent | openstack | :-)   | True           |
   +--------------------------------------+--------------------+-----------+-------+----------------+

* 将router_admin设置为L3代理类型::

   # quantum l3-agent-router-add 2b68d118-c4bb-44a0-8387-678c5bdb1653 router_admin

   Added router router_admin to L3 agent

* 将net_admin子网与router_admin路由关联::

   # quantum router-interface-add 76d8ac10-a6df-4dfa-b691-297da374c811 fb141492-8aa1-437b-8192-315e19e7f4d2

   Added interface to router 76d8ac10-a6df-4dfa-b691-297da374c811

* 创建外网net_external，注意设置--router:external=True::

   # quantum net-create net_external --router:external=True --shared

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 7a7acad8-cabf-49f8-804f-ce6871d9cd63 |
   | name                      | net_external                         |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 2                                    |
   | router:external           | True                                 |
   | shared                    | True                                 |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | 8c0104041b034df3a79c17a9517dd3f9     |
   +---------------------------+--------------------------------------+

* 为net_external创建子网，注意设置的gateway必须在给到的网段内::

   # quantum subnet-create net_external --gateway 192.168.100.1 192.168.100.0/24 --enable_dhcp=False

   Created a new subnet:
   +------------------+------------------------------------------------------+
   | Field            | Value                                                |
   +------------------+------------------------------------------------------+
   | allocation_pools | {"start": "192.168.100.2", "end": "192.168.100.254"} |
   | cidr             | 192.168.100.0/24                                     |
   | dns_nameservers  |                                                      |
   | enable_dhcp      | False                                                |
   | gateway_ip       | 192.168.100.1                                        |
   | host_routes      |                                                      |
   | id               | 837ad514-3c05-4357-9a36-0b18adcfb354                 |
   | ip_version       | 4                                                    |
   | name             |                                                      |
   | network_id       | 7a7acad8-cabf-49f8-804f-ce6871d9cd63                 |
   | tenant_id        | 8c0104041b034df3a79c17a9517dd3f9                     |
   +------------------+------------------------------------------------------+

* 将net_external与router_admin路由器关联::

   # quantum router-gateway-set router_admin net_external

   Set gateway for router router_admin

* 创建floating ip::

   # quantum floatingip-create net_external

   Created a new floatingip:
   +---------------------+--------------------------------------+
   | Field               | Value                                |
   +---------------------+--------------------------------------+
   | fixed_ip_address    |                                      |
   | floating_ip_address | 192.168.100.3                        |
   | floating_network_id | 7a7acad8-cabf-49f8-804f-ce6871d9cd63 |
   | id                  | 15bb69fa-972d-4e86-91fc-250dc1b20fe2 |
   | port_id             |                                      |
   | router_id           |                                      |
   | tenant_id           | 8c0104041b034df3a79c17a9517dd3f9     |
   +---------------------+--------------------------------------+

   # quantum floatingip-create net_external

   Created a new floatingip:
   +---------------------+--------------------------------------+
   | Field               | Value                                |
   +---------------------+--------------------------------------+
   | fixed_ip_address    |                                      |
   | floating_ip_address | 192.168.100.4                        |
   | floating_network_id | 7a7acad8-cabf-49f8-804f-ce6871d9cd63 |
   | id                  | 561e3530-d543-427f-986a-aaff64cb1a87 |
   | port_id             |                                      |
   | router_id           |                                      |
   | tenant_id           | 8c0104041b034df3a79c17a9517dd3f9     |
   +---------------------+--------------------------------------+

* 为admin租户创建虚拟机并关联floating ip(可通过web界面创建虚拟机并关联floating ip)::

   注意：如下生成秘钥对，并上传ssh公钥：
   # ssh-keygen
   Generating public/private rsa key pair.
   Enter file in which to save the key (/root/.ssh/id_rsa):
   Created directory '/root/.ssh'.
   Enter passphrase (empty for no passphrase):
   Enter same passphrase again:
   Your identification has been saved in /root/.ssh/id_rsa.
   Your public key has been saved in /root/.ssh/id_rsa.pub.
   The key fingerprint is:
   ab:dc:48:ae:a6:12:d5:8b:db:cf:7c:31:c1:4a:03:39 root@grizzly
   The key's randomart image is:
   +--[ RSA 2048]----+
   |     .           |
   |    E            |
   |   . o .         |
   |  . . o o        |
   | . . o oS.       |
   |. . . . o.       |
   | . o  . .o       |
   |. . o* +.        |
   | ..o.oO..        |
   +-----------------+

   # nova keypair-add --pub_key /root/.ssh/id_rsa.pub nova-key

   上传公钥后便可以通过 ssh -i /root/.ssh/id_rsa cirros@192.168.100.3 登陆cirros虚拟机。

   # nova list

   +--------------------------------------+-----------------+--------+---------------------------------------+
   | ID                                   | Name            | Status | Networks                              |
   +--------------------------------------+-----------------+--------+---------------------------------------+
   | fb4c93a0-fc83-4779-b85f-d7326c238c94 | ubuntu.vm.admin | ACTIVE | net_admin=172.16.100.4, 192.168.100.4 |
   | 5b918d39-1ac9-4a76-83d5-8b32a29ed3fe | vm.admin        | ACTIVE | net_admin=172.16.100.3, 192.168.100.3 |
   +--------------------------------------+-----------------+--------+---------------------------------------+


9.1. 创建leju.com租户、内网、路由器和虚拟机并关联外网
------------------

* 创建leju.com租户::

   # keystone tenant-create --name leju.com

   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |                                  |
   |   enabled   |               True               |
   |      id     | 5585ffbad86d495d88b5f95729b1dc60 |
   |     name    |             leju.com             |
   +-------------+----------------------------------+

* 在leju.com租户中创建dongliang用户::

   # keystone user-create --name=dongliang --pass=123456 --tenant-id=5585ffbad86d495d88b5f95729b1dc60 --email=dongliang@leju.com

   +----------+----------------------------------+
   | Property |              Value               |
   +----------+----------------------------------+
   |  email   |        dongliang@leju.com        |
   | enabled  |               True               |
   |    id    | 21efde97763147718fee478634cd3e70 |
   |   name   |            dongliang             |
   | tenantId | 5585ffbad86d495d88b5f95729b1dc60 |
   +----------+----------------------------------+

* 列出预定义的角色::

   # keystone role-list

   +----------------------------------+----------------------+
   |                id                |         name         |
   +----------------------------------+----------------------+
   | b90f2f8f84c4454f800f053dd5b6a54e |    KeystoneAdmin     |
   | 0ba9be2eb2c145ffb90def5a75646ed2 | KeystoneServiceAdmin |
   | b7e97eecf8cd4d6aa6f4091206ad6282 |        Member        |
   | 9fe2ff9ee4384b1894a90878d3e92bab |       _member_       |
   | 47eda7948e5d430bad3ce937fb00dc3b |        admin         |
   +----------------------------------+----------------------+

* 为用户dongliang添加角色::

   # keystone user-role-add --tenant-id 5585ffbad86d495d88b5f95729b1dc60 --user-id 21efde97763147718fee478634cd3e70 --role-id 47eda7948e5d430bad3ce937fb00dc3b

* 为leju.com租户创建网络::

   # quantum net-create --tenant-id 5585ffbad86d495d88b5f95729b1dc60 net_leju_com

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 599e5a95-ff7f-49e5-9930-03e99e3a2d8d |
   | name                      | net_leju_com                         |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 3                                    |
   | router:external           | False                                |
   | shared                    | False                                |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | 5585ffbad86d495d88b5f95729b1dc60     |
   +---------------------------+--------------------------------------+

* 为leju.com租户创建子网::

   # quantum subnet-create --tenant-id 5585ffbad86d495d88b5f95729b1dc60 net_leju_com 172.16.200.0/24

   Created a new subnet:
   +------------------+----------------------------------------------------+
   | Field            | Value                                              |
   +------------------+----------------------------------------------------+
   | allocation_pools | {"start": "172.16.200.2", "end": "172.16.200.254"} |
   | cidr             | 172.16.200.0/24                                    |
   | dns_nameservers  |                                                    |
   | enable_dhcp      | True                                               |
   | gateway_ip       | 172.16.200.1                                       |
   | host_routes      |                                                    |
   | id               | dbb59749-8f05-474d-b26d-745254a22669               |
   | ip_version       | 4                                                  |
   | name             |                                                    |
   | network_id       | 599e5a95-ff7f-49e5-9930-03e99e3a2d8d               |
   | tenant_id        | 5585ffbad86d495d88b5f95729b1dc60                   |
   +------------------+----------------------------------------------------+

* 为leju.com租户创建路由器::

   # quantum router-create --tenant-id 5585ffbad86d495d88b5f95729b1dc60 router_leju_com

   Created a new router:
   +-----------------------+--------------------------------------+
   | Field                 | Value                                |
   +-----------------------+--------------------------------------+
   | admin_state_up        | True                                 |
   | external_gateway_info |                                      |
   | id                    | 451a6166-d082-4f02-8f37-07703a8118ab |
   | name                  | router_leju_com                      |
   | status                | ACTIVE                               |
   | tenant_id             | 5585ffbad86d495d88b5f95729b1dc60     |
   +-----------------------+--------------------------------------+

* 列出代理列表::

   quantum agent-list

   +--------------------------------------+--------------------+-----------+-------+----------------+
   | id                                   | agent_type         | host      | alive | admin_state_up |
   +--------------------------------------+--------------------+-----------+-------+----------------+
   | 2b68d118-c4bb-44a0-8387-678c5bdb1653 | L3 agent           | openstack | :-)   | True           |
   | 7b42460c-cffd-494f-94b1-c6b4f3b5e102 | DHCP agent         | openstack | :-)   | True           |
   | e443fbf2-398c-47ab-89d9-5d9907217379 | Open vSwitch agent | openstack | :-)   | True           |
   +--------------------------------------+--------------------+-----------+-------+----------------+

* 设置路由器使用L3代理::

   # quantum l3-agent-router-add 2b68d118-c4bb-44a0-8387-678c5bdb1653 router_leju_com

   Added router router_leju_com to L3 agent

* 连接net_leju_com到router_leju_com::

   # quantum router-interface-add 451a6166-d082-4f02-8f37-07703a8118ab dbb59749-8f05-474d-b26d-745254a22669

   Added interface to router 451a6166-d082-4f02-8f37-07703a8118ab

* 设置net_leju_com外网网关::

   # quantum router-gateway-set 451a6166-d082-4f02-8f37-07703a8118ab net_external

   Set gateway for router 451a6166-d082-4f02-8f37-07703a8118ab

* 设置leju.com租户环境变量::

   # cat creds-dongliang

   export OS_TENANT_NAME=leju.com
   export OS_USERNAME=dongliang
   export OS_PASSWORD=123456
   export OS_AUTH_URL="http://192.168.100.51:5000/v2.0/"

* 用dongliang用户登陆web界面，创建虚拟主机vm.leju.com

* 使变量生效::

   source creds-dongliang

* 列出虚拟主机::

   # nova list

   +--------------------------------------+-------------+--------+---------------------------+
   | ID                                   | Name        | Status | Networks                  |
   +--------------------------------------+-------------+--------+---------------------------+
   | eefc20a9-251c-44de-99ee-179463cb7aca | vm.leju.com | ACTIVE | net_leju_com=172.16.200.2 |
   +--------------------------------------+-------------+--------+---------------------------+

* 列出vm.leju.com虚拟机的端口::

   # quantum port-list -- --device_id eefc20a9-251c-44de-99ee-179463cb7aca

   +--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+
   | id                                   | name | mac_address       | fixed_ips                                                                           |
   +--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+
   | d0195246-5863-4ede-ac40-3cc06516279e |      | fa:16:3e:0c:f2:01 | {"subnet_id": "dbb59749-8f05-474d-b26d-745254a22669", "ip_address": "172.16.200.2"} |
   +--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+

* 为vm.leju.com创建floating ip::

   # quantum floatingip-create net_external

   Created a new floatingip:
   +---------------------+--------------------------------------+
   | Field               | Value                                |
   +---------------------+--------------------------------------+
   | fixed_ip_address    |                                      |
   | floating_ip_address | 192.168.100.8                        |
   | floating_network_id | 7a7acad8-cabf-49f8-804f-ce6871d9cd63 |
   | id                  | 2efa6e49-9d99-4402-9a61-85c235d0ccb8 |
   | port_id             |                                      |
   | router_id           |                                      |
   | tenant_id           | 5585ffbad86d495d88b5f95729b1dc60     |
   +---------------------+--------------------------------------+

* 将新创建的floating ip与vm.leju.com关联::

   # quantum floatingip-associate 2efa6e49-9d99-4402-9a61-85c235d0ccb8 d0195246-5863-4ede-ac40-3cc06516279e

   Associated floatingip 2efa6e49-9d99-4402-9a61-85c235d0ccb8




