==========================================================
  OpenStack Grizzly 安装指南
==========================================================

:Version: 1.0
:Source: https://github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN
:Keywords: 多点OpenStack安装, Grizzly, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.04 (64 bits).

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
  1. 环境搭建
  2. 控制节点
  3. 计算和网络节点
  4. OpenStack使用
  5. 参考文档


0. 简介
==============

OpenStack Grizzly安装指南旨在让你轻松创建自己的OpenStack云平台。

状态: Stable


1. 环境搭建
====================

:节点角色: NICs
:控制节点: eth0 (10.10.10.51), eth1 (192.168.100.51)
:计算节点1: eth0 (10.10.10.52), eth1 (10.20.20.52), eth2 (192.168.100.52)
:计算节点2: eth0 (10.10.10.53), eth1 (10.20.20.53)

**注意1:** 你总是可以使用dpkg -s <packagename>确认你使用的是grizzly软件包(版本: 2013.1)

**注意2:** 这个是当前网络架构

.. image:: http://i.imgur.com/iOyv0Wn.jpg

2. 控制节点
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
   address 10.10.10.51
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

* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p

2.3. 安装MySQL
------------

* 安装MySQL并为root用户设置密码::

   apt-get install -y mysql-server python-mysqldb

* 配置mysql监听所有网络接口请求::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

2.4. 安装RabbitMQ和NTP
------------

* 安装RabbitMQ::

   apt-get install -y rabbitmq-server 

* 安装NTP服务::

   apt-get install -y ntp

2.5. 创建数据库
------------

* 创建数据库::

   mysql -u root -p
   
   #Keystone
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   
   #Glance
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';

   #Quantum
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';

   #Nova
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';      

   #Cinder
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';

   quit;

2.6. 配置Keystone
------------

* 安装keystone软件包::

   apt-get install -y keystone

* 在/etc/keystone/keystone.conf中设置连接到新创建的数据库::

   connection = mysql://keystoneUser:keystonePass@10.10.10.51/keystone

* 重启身份认证服务并同步数据库::

   service keystone restart
   keystone-manage db_sync

* 使用git仓库中脚本填充keystone数据库： `脚本文件夹 <https://github.com/ist0ne/OpenStack-Grizzly-Install-Guide/tree/master/KeystoneScripts>`_ ::

   #注意在执行脚本前请按你的网卡配置修改HOST_IP和HOST_IP_EXT

   wget https://raw.github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN/OVS_MutliNode/KeystoneScripts/keystone_basic.sh
   wget https://raw.github.com/ist0ne/OpenStack-Grizzly-Install-Guide-CN/OVS_MutliNode/KeystoneScripts/keystone_endpoints_basic.sh

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

2.7. 设置Glance
------------

* 安装Glance::

   apt-get install -y glance

* 按下面更新/etc/glance/glance-api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   delay_auth_decision = true
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 按下面更新/etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 按下面更新/etc/glance/glance-api.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.10.51/glance

* 和::

   [paste_deploy]
   flavor = keystone
   
* 按下面更新/etc/glance/glance-registry.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.10.51/glance

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

2.8. 设置Quantum
------------

* 安装Quantum组件::

   apt-get install -y quantum-server

* 编辑/etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑OVS配置文件/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@10.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True

   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 编辑/etc/quantum/quantum.conf::

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

* 重启quantum所有服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

2.9. 设置Nova
------------------

* 安装nova组件::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
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
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.51
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://10.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Quantum + Nova Security groups
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=quantum
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   #-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   #Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900
 
* 同步数据库::

   nova-manage db sync

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list

2.10. 设置Cinder
------------------

* 安装软件包::

   apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* 配置iscsi服务::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* 重启服务::
   
   service iscsitarget start
   service open-iscsi start

* 如下配置/etc/cinder/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 192.168.100.51
   service_port = 5000
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* 编辑/etc/cinder/cinder.conf::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.10.10.51/cinder
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

* 重启cinder服务::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done

* 确认cinder服务在运行::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done

2.11. 设置Horizon
------------------

* 如下安装horizon ::

   apt-get install -y openstack-dashboard memcached

* 如果你不喜欢OpenStack ubuntu主题, 你可以停用它::

   dpkg --purge openstack-dashboard-ubuntu-theme

* 重启Apache和memcached服务::

   service apache2 restart; service memcached restart

3. 所有计算和网络节点
================

3.1. 准备节点
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

* 安装ntp服务::

   apt-get install -y ntp

* 配置ntp服务从控制节点同步时间::

   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the network node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 10.10.10.51/g' /etc/ntp.conf

   service ntp restart

3.2. 配置网络
-----------------

* 计算节点1网卡如下设置::

   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 10.10.10.52
   netmask 255.255.255.0

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 10.20.20.52
   netmask 255.255.255.0

   # VM internet Access
   auto eth2
   iface eth2 inet static
   address 192.168.100.52
   netmask 255.255.255.0

* 计算节点2网卡如下设置::

   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 10.10.10.53
   netmask 255.255.255.0

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 10.20.20.53
   netmask 255.255.255.0

   # VM internet Access
   auto eth2
   iface eth2 inet static
   address 192.168.100.53
   netmask 255.255.255.0

* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p


3.3. OpenVSwitch
------------

* 安装OpenVSwitch软件包::

   apt-get install -y openvswitch-controller openvswitch-switch openvswitch-brcompat

* 修改openvswitch-switch配置文件::

   sed -i 's/# BRCOMPAT=no/BRCOMPAT=yes/g' /etc/default/openvswitch-switch

* 重启openvswitch-switch（注意ovs-brcompatd是否启动，如果未启动需要强制加载）::

   /etc/init.d/openvswitch-switch restart

* 如果有bridge module is loaded, not loading brcompat提示，需要先卸载bridge模块::

   lsmod |grep bridge
   rmmod bridge

* 强制加载brcompat内核模块::

   /etc/init.d/openvswitch-switch force-reload-kmod

* 查看ovs-brcompatd、ovs-vswitchd、ovsdb-server是否均已启动::

   /etc/init.d/openvswitch-switch restart

* 查看brcompat内核模块已挂载::

   lsmod | grep brcompat

   brcompat               13513  0
   openvswitch            84124  1 brcompat

* 如果还是有问题执行下面步骤，直到ovs-brcompatd、ovs-vswitchd、ovsdb-server都启动::

   root@openstack:~# apt-get install -y openvswitch-datapath-source
   root@openstack:~# module-assistant auto-install openvswitch-datapath
   root@openstack:~# /etc/init.d/openvswitch-switch force-reload-kmod
   root@openstack:~# /etc/init.d/openvswitch-switch restart

   文档参考：http://blog.scottlowe.org/2012/08/17/installing-kvm-and-open-vswitch-on-ubuntu/

* 添加网桥 br-ex 并把网卡 eth1 加入 br-ex::

   ovs-vsctl  add-br br-ex
   ovs-vsctl add-port br-ex eth2

* 如下编辑/etc/network/interfaces::

   # This file describes the network interfaces available on your system
   # and how to activate them. For more information, see interfaces(5).

   # The loopback network interface
   auto lo
   iface lo inet loopback

   # Not internet connected(used for OpenStack management)
   # The primary network interface
   auto eth0
   iface eth0 inet static
   # This is an autoconfigured IPv6 interface
   # iface eth0 inet6 auto
   address 10.10.10.52    # 计算节点2改为10.10.10.53
   netmask 255.255.255.0

   auto eth1
   iface eth1 inet static
   address 10.20.20.52    # 计算节点2改为10.10.10.53
   netmask 255.255.255.0

   #For Exposing OpenStack API over the internet
   auto eth2
   iface eth2 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

   auto br-ex
   iface br-ex inet static
   address 192.168.100.52    # 计算节点2改为10.10.10.53
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

* 重启网络服务::

   /etc/init.d/networking restart

* 创建内网网桥br-int::

   ovs-vsctl add-br br-int

* 查看网桥配置::

   root@openstack-network:~# ovs-vsctl list-br
   br-ex
   br-int

   root@openstack-network:~# ovs-vsctl show
   ebea0b50-e450-41ea-babb-a094ca8d69fa
       Bridge br-int
           Port br-int
               Interface br-int
                   type: internal
       Bridge br-ex
           Port "eth2"
               Interface "eth2"
           Port br-ex
               Interface br-ex
                   type: internal
       ovs_version: "1.4.0+build0"

3.4. Quantum-*
------------

* 安装Quantum组件::

   apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent quantum-metadata-agent

* 编辑/etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑OVS配置文件/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@10.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   enable_tunneling = True
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.10.10.52    # 计算节点2改为10.10.10.53

   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 更新/etc/quantum/metadata_agent.ini::

   # The Quantum user information for accessing the Quantum API.
   auth_url = http://10.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

   # IP address used by Nova metadata server
   nova_metadata_ip = 10.10.10.51

   # TCP Port used by Nova metadata server
   nova_metadata_port = 8775

   metadata_proxy_shared_secret = helloOpenStack

* 编辑/etc/quantum/quantum.conf::

   # 确保RabbitMQ IP指向了控制节点
   rabbit_host = 10.10.10.51

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

* 编辑/etc/quantum/l3_agent.ini::

   [DEFAULT]
   interface_driver = quantum.agent.linux.interface.OVSInterfaceDriver
   use_namespaces = True
   external_network_bridge = br-ex
   signing_dir = /var/cache/quantum
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   auth_url = http://10.10.10.51:35357/v2.0
   l3_agent_manager = quantum.agent.l3_agent.L3NATAgentWithStateReport
   root_helper = sudo quantum-rootwrap /etc/quantum/rootwrap.conf
   interface_driver = quantum.agent.linux.interface.OVSInterfaceDriver
   enable_multi_host = True    # 开启多主机模式

* 编辑/etc/quantum/dhcp_agent.ini::

   [DEFAULT]
   interface_driver = quantum.agent.linux.interface.OVSInterfaceDriver
   dhcp_driver = quantum.agent.linux.dhcp.Dnsmasq
   use_namespaces = True
   signing_dir = /var/cache/quantum
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   auth_url = http://10.10.10.51:35357/v2.0
   dhcp_agent_manager = quantum.agent.dhcp_agent.DhcpAgentWithStateReport
   root_helper = sudo quantum-rootwrap /etc/quantum/rootwrap.conf
   state_path = /var/lib/quantum

   enable_multi_host = True    # 开启多主机模式

   # The DHCP server can assist with providing metadata support on isolated
   # networks. Setting this value to True will cause the DHCP server to append
   # specific host routes to the DHCP request.  The metadata service will only
   # be activated when the subnet gateway_ip is None.  The guest instance must
   # be configured to request host routes via DHCP (Option 121).
   enable_isolated_metadata = False

   # Allows for serving metadata requests coming from a dedicated metadata
   # access network whose cidr is 169.254.169.254/16 (or larger prefix), and
   # is connected to a Quantum router from which the VMs send metadata
   # request. In this case DHCP Option 121 will not be injected in VMs, as
   # they will be able to reach 169.254.169.254 through a router.
   # This option requires enable_isolated_metadata = True
   enable_metadata_network = False

* 重启quantum所有服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

3.5. KVM
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

* 删除默认的虚拟网桥::

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

3.6. Nova
------------------

* 安装nova组件::

   apt-get install -y nova-compute-kvm

   注意：如果你的宿主机不支持kvm虚拟化，可把nova-compute-kvm换成nova-compute-qemu
   同时/etc/nova/nova-compute.conf配置文件中的libvirt_type=qemu

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
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
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.52    # 计算节点二改为10.10.10.53
   vncserver_listen=0.0.0.0
   
   # Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack
   
   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://10.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://10.10.10.51:35357/v2.0
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
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list

4. OpenStack使用
================

网络拓扑如下：

.. image:: http://i.imgur.com/WdRDVZJ.png

5.1. 为admin租户创建内网、外网、路由器和虚拟机
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

   +----------------------------------+---------+---------+--------------------+
   |                id                |   name  | enabled |       email        |
   +----------------------------------+---------+---------+--------------------+
   | 1ec119f9c8f14b8fa5cbe80395017462 |  admin  |   True  |  admin@domain.com  |
   | 3c732419e41f401ab8b38ba4fd794c24 |  cinder |   True  | cinder@domain.com  |
   | 1cce810d65d6498ea6a167e612e75bde |  glance |   True  | glance@domain.com  |
   | 3cd285e00789485c87b34c0b039816f9 |   nova  |   True  |  nova@domain.com   |
   | e65a97a59a5140f39787ae62d9fb42a7 | quantum |   True  | quantum@domain.com |
   +----------------------------------+---------+---------+--------------------+

* 列出已创建的租户::

   # keystone tenant-list

   +----------------------------------+---------+---------+
   |                id                |   name  | enabled |
   +----------------------------------+---------+---------+
   | d2d70c131e86453f8296940da08bb574 |  admin  |   True  |
   | 8a82c60ef6544e648c1cf7b19212c898 | service |   True  |
   +----------------------------------+---------+---------+

* 为admin租户创建网络::

   # quantum net-create --tenant-id d2d70c131e86453f8296940da08bb574 net_admin

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 99816d06-0ecf-4d1f-a2fa-e46924b477b6 |
   | name                      | net_admin                            |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 1                                    |
   | router:external           | False                                |
   | shared                    | False                                |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | d2d70c131e86453f8296940da08bb574     |
   +---------------------------+--------------------------------------+

# 为admin租户创建子网::

   # quantum subnet-create --tenant-id d2d70c131e86453f8296940da08bb574 net_admin 172.16.100.0/24

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
   | id               | 756f203f-8fd3-4074-9a12-1328cfbc41bf               |
   | ip_version       | 4                                                  |
   | name             |                                                    |
   | network_id       | 99816d06-0ecf-4d1f-a2fa-e46924b477b6               |
   | tenant_id        | d2d70c131e86453f8296940da08bb574                   |
   +------------------+----------------------------------------------------+

* 为admin租户创建路由器::

   # quantum router-create --tenant-id d2d70c131e86453f8296940da08bb574 router_admin

   Created a new router:
   +-----------------------+--------------------------------------+
   | Field                 | Value                                |
   +-----------------------+--------------------------------------+
   | admin_state_up        | True                                 |
   | external_gateway_info |                                      |
   | id                    | 813eb696-58e3-4721-b6b2-d7d1f946502c |
   | name                  | router_admin                         |
   | status                | ACTIVE                               |
   | tenant_id             | d2d70c131e86453f8296940da08bb574     |
   +-----------------------+--------------------------------------+

* 列出路由代理类型::

   # quantum agent-list

   +--------------------------------------+--------------------+----------+-------+----------------+
   | id                                   | agent_type         | host     | alive | admin_state_up |
   +--------------------------------------+--------------------+----------+-------+----------------+
   | 03ad5d83-d089-4664-ba65-5d53970c5a1e | DHCP agent         | Compute1 | :-)   | True           |
   | 071b8408-74fa-43bc-a3d4-68ab0d42796c | L3 agent           | Compute1 | :-)   | True           |
   | 2be821e0-9629-4d9b-8b50-79e5237278ed | Open vSwitch agent | Compute1 | :-)   | True           |
   | 5b8de451-0cbc-4637-9070-51b8e9a4b8d8 | L3 agent           | Compute2 | :-)   | True           |
   | 883c97a0-ac6b-418c-8790-e80b6c177d70 | DHCP agent         | Compute2 | :-)   | True           |
   | f353ea02-48a8-4eee-98b8-427a67888962 | Open vSwitch agent | Compute2 | :-)   | True           |
   +--------------------------------------+--------------------+----------+-------+----------------+

* 将router_admin设置为L3代理类型（将router_admin与Compute1的L3代理关联）::

   # quantum quantum l3-agent-router-add 071b8408-74fa-43bc-a3d4-68ab0d42796c router_admin

   Added router router_admin to L3 agent

* 将net_admin子网与router_admin路由关联::

   # quantum router-interface-add 813eb696-58e3-4721-b6b2-d7d1f946502c 756f203f-8fd3-4074-9a12-1328cfbc41bf

   Added interface to router 813eb696-58e3-4721-b6b2-d7d1f946502c

* 创建外网net_external，注意设置--router:external=True::

   # quantum net-create net_external --router:external=True --shared

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | 750119bd-3246-4179-a4e9-bdfade8fb88a |
   | name                      | net_external                         |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 2                                    |
   | router:external           | True                                 |
   | shared                    | True                                 |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | d2d70c131e86453f8296940da08bb574     |
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
   | id               | 53424a33-e685-469e-b529-eccf75504ba1                 |
   | ip_version       | 4                                                    |
   | name             |                                                      |
   | network_id       | 750119bd-3246-4179-a4e9-bdfade8fb88a                 |
   | tenant_id        | d2d70c131e86453f8296940da08bb574                     |
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
   | floating_network_id | 750119bd-3246-4179-a4e9-bdfade8fb88a |
   | id                  | c9904183-6b14-426f-8a23-c4269be933a5 |
   | port_id             |                                      |
   | router_id           |                                      |
   | tenant_id           | d2d70c131e86453f8296940da08bb574     |
   +---------------------+--------------------------------------+

   # quantum floatingip-create net_external

   Created a new floatingip:
   +---------------------+--------------------------------------+
   | Field               | Value                                |
   +---------------------+--------------------------------------+
   | fixed_ip_address    |                                      |
   | floating_ip_address | 192.168.100.4                        |
   | floating_network_id | 750119bd-3246-4179-a4e9-bdfade8fb88a |
   | id                  | 0be595f6-ef6f-4257-a3ee-c3b2e951a397 |
   | port_id             |                                      |
   | router_id           |                                      |
   | tenant_id           | d2d70c131e86453f8296940da08bb574     |
   +---------------------+--------------------------------------+

* 运行虚拟机通过22端口被访问并能被ping通::

   # nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

   +-------------+-----------+---------+-----------+--------------+
   | IP Protocol | From Port | To Port | IP Range  | Source Group |
   +-------------+-----------+---------+-----------+--------------+
   | tcp         | 22        | 22      | 0.0.0.0/0 |              |
   +-------------+-----------+---------+-----------+--------------+

   # nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

   +-------------+-----------+---------+-----------+--------------+
   | IP Protocol | From Port | To Port | IP Range  | Source Group |
   +-------------+-----------+---------+-----------+--------------+
   | icmp        | -1        | -1      | 0.0.0.0/0 |              |
   +-------------+-----------+---------+-----------+--------------+

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


5.2. 创建leju.com租户、内网、路由器和虚拟机并关联外网
------------------

* 创建leju.com租户::

   # keystone tenant-create --name leju.com

   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |                                  |
   |   enabled   |               True               |
   |      id     | f1ee07a9fdd740d78c71d6fa21537f9a |
   |     name    |             leju.com             |
   +-------------+----------------------------------+

* 在leju.com租户中创建dongliang用户::

   # keystone user-create --name=dongliang --pass=123456 --tenant-id f1ee07a9fdd740d78c71d6fa21537f9a --email=dongliang@leju.com

   +----------+----------------------------------+
   | Property |              Value               |
   +----------+----------------------------------+
   |  email   |        dongliang@leju.com        |
   | enabled  |               True               |
   |    id    | 149705e3e9db4cfbb4593e60cd3c3a82 |
   |   name   |            dongliang             |
   | tenantId | f1ee07a9fdd740d78c71d6fa21537f9a |
   +----------+----------------------------------+

* 列出预定义的角色::

   # keystone role-list

   +----------------------------------+----------------------+
   |                id                |         name         |
   +----------------------------------+----------------------+
   | 1105a8ced2a54be1a9e69ef019963ba0 |    KeystoneAdmin     |
   | 717df1c9ddb641f9b0fb9195a4453608 | KeystoneServiceAdmin |
   | e651a0e1d19a4c87a2bbc0d3d14df4af |        Member        |
   | 9fe2ff9ee4384b1894a90878d3e92bab |       _member_       |
   | 64ee3ca0ff6a4e1c89cd73b2a8b15a32 |        admin         |
   +----------------------------------+----------------------+

* 为用户dongliang添加角色::

   # keystone user-role-add --tenant-id f1ee07a9fdd740d78c71d6fa21537f9a --user-id 149705e3e9db4cfbb4593e60cd3c3a82 --role-id 64ee3ca0ff6a4e1c89cd73b2a8b15a32

* 为leju.com租户创建网络::

   # quantum net-create --tenant-id f1ee07a9fdd740d78c71d6fa21537f9a net_leju_com

   Created a new network:
   +---------------------------+--------------------------------------+
   | Field                     | Value                                |
   +---------------------------+--------------------------------------+
   | admin_state_up            | True                                 |
   | id                        | bcb7cebf-bc0b-496c-94ed-1c7c96ae94fd |
   | name                      | net_leju_com                         |
   | provider:network_type     | gre                                  |
   | provider:physical_network |                                      |
   | provider:segmentation_id  | 3                                    |
   | router:external           | False                                |
   | shared                    | False                                |
   | status                    | ACTIVE                               |
   | subnets                   |                                      |
   | tenant_id                 | f1ee07a9fdd740d78c71d6fa21537f9a     |
   +---------------------------+--------------------------------------+

* 为leju.com租户创建子网::

   # quantum subnet-create --tenant-id f1ee07a9fdd740d78c71d6fa21537f9a net_leju_com 172.16.200.0/24

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
   | id               | b1085543-3a4f-4965-ade4-e3b06d89a285               |
   | ip_version       | 4                                                  |
   | name             |                                                    |
   | network_id       | bcb7cebf-bc0b-496c-94ed-1c7c96ae94fd               |
   | tenant_id        | f1ee07a9fdd740d78c71d6fa21537f9a                   |
   +------------------+----------------------------------------------------+

* 为leju.com租户创建路由器::

   # quantum router-create --tenant-id f1ee07a9fdd740d78c71d6fa21537f9a router_leju_com

   Created a new router:
   +-----------------------+--------------------------------------+
   | Field                 | Value                                |
   +-----------------------+--------------------------------------+
   | admin_state_up        | True                                 |
   | external_gateway_info |                                      |
   | id                    | 9b8ee7f4-a3b4-41e2-a28e-4feca3ba1389 |
   | name                  | router_leju_com                      |
   | status                | ACTIVE                               |
   | tenant_id             | f1ee07a9fdd740d78c71d6fa21537f9a     |
   +-----------------------+--------------------------------------+

* 列出代理列表::

   # quantum agent-list

   +--------------------------------------+--------------------+----------+-------+----------------+
   | id                                   | agent_type         | host     | alive | admin_state_up |
   +--------------------------------------+--------------------+----------+-------+----------------+
   | 03ad5d83-d089-4664-ba65-5d53970c5a1e | DHCP agent         | Compute1 | :-)   | True           |
   | 071b8408-74fa-43bc-a3d4-68ab0d42796c | L3 agent           | Compute1 | :-)   | True           |
   | 2be821e0-9629-4d9b-8b50-79e5237278ed | Open vSwitch agent | Compute1 | :-)   | True           |
   | 5b8de451-0cbc-4637-9070-51b8e9a4b8d8 | L3 agent           | Compute2 | :-)   | True           |
   | 883c97a0-ac6b-418c-8790-e80b6c177d70 | DHCP agent         | Compute2 | :-)   | True           |
   | f353ea02-48a8-4eee-98b8-427a67888962 | Open vSwitch agent | Compute2 | :-)   | True           |
   +--------------------------------------+--------------------+----------+-------+----------------+

* 设置路由器使用L3代理(将router_leju_com与Compute2的L3代理相关联)::

   # quantum l3-agent-router-add 5b8de451-0cbc-4637-9070-51b8e9a4b8d8 router_leju_com

   Added router router_leju_com to L3 agent

* 连接net_leju_com到router_leju_com::

   # quantum router-interface-add 9b8ee7f4-a3b4-41e2-a28e-4feca3ba1389 b1085543-3a4f-4965-ade4-e3b06d89a285

   Added interface to router 9b8ee7f4-a3b4-41e2-a28e-4feca3ba1389

* 设置net_leju_com外网网关::

   # quantum router-gateway-set  9b8ee7f4-a3b4-41e2-a28e-4feca3ba1389 net_external

   Set gateway for router 9b8ee7f4-a3b4-41e2-a28e-4feca3ba1389

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
   | d0195246-5863-4ede-ac40-3cc06516279e |      | fa:16:3e:0c:f2:01 | {"subnet_id": "b1085543-3a4f-4965-ade4-e3b06d89a285", "ip_address": "172.16.200.2"} |
   +--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+

* 为vm.leju.com创建floating ip::

   # quantum floatingip-create net_external

   Created a new floatingip:
   +---------------------+--------------------------------------+
   | Field               | Value                                |
   +---------------------+--------------------------------------+
   | fixed_ip_address    |                                      |
   | floating_ip_address | 192.168.100.8                        |
   | floating_network_id | b1085543-3a4f-4965-ade4-e3b06d89a285 |
   | id                  | 2efa6e49-9d99-4402-9a61-85c235d0ccb8 |
   | port_id             |                                      |
   | router_id           |                                      |
   | tenant_id           | f1ee07a9fdd740d78c71d6fa21537f9a     |
   +---------------------+--------------------------------------+

* 将新创建的floating ip与vm.leju.com关联::

   # quantum floatingip-associate 2efa6e49-9d99-4402-9a61-85c235d0ccb8 d0195246-5863-4ede-ac40-3cc06516279e

   Associated floatingip 2efa6e49-9d99-4402-9a61-85c235d0ccb8

6. 参考文档
================

`Boostrapping Open vSwitch and Quantum <https://a248.e.akamai.net/cdn.hpcloudsvc.com/h9f25be84b35c201beea6b13c85876258/prodaw2/Bootstrapping_OVS_Quantum--final_20130319.html>`_

`Cisco OpenStack Edition: Folsom Manual Install <http://docwiki.cisco.com/wiki/Cisco_OpenStack_Edition:_Folsom_Manual_Install>`_


