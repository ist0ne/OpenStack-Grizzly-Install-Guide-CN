网络配置如下：
root@openstack:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 00:1c:42:c7:d2:f5
          inet addr:10.10.100.51  Bcast:10.10.100.255  Mask:255.255.255.0
          inet6 addr: fdb2:2c26:f4e4:1:b990:97f:44f4:a7b5/64 Scope:Global
          inet6 addr: fdb2:2c26:f4e4:1:21c:42ff:fec7:d2f5/64 Scope:Global
          inet6 addr: fe80::21c:42ff:fec7:d2f5/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:18 errors:0 dropped:0 overruns:0 frame:0
          TX packets:17 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2767 (2.7 KB)  TX bytes:3153 (3.1 KB)

eth1      Link encap:Ethernet  HWaddr 00:1c:42:7b:0b:08
          inet addr:192.168.200.51  Bcast:192.168.200.255  Mask:255.255.255.0
          inet6 addr: fdb2:2c26:f4e4:0:165:aaf4:d6ee:573f/64 Scope:Global
          inet6 addr: fdb2:2c26:f4e4:0:21c:42ff:fe7b:b08/64 Scope:Global
          inet6 addr: fe80::21c:42ff:fe7b:b08/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:161 errors:0 dropped:0 overruns:0 frame:0
          TX packets:123 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:17341 (17.3 KB)  TX bytes:14301 (14.3 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

# 安装openvswitch（OVS）
root@openstack:~# apt-get install openvswitch-controller openvswitch-switch openvswitch-brcompat

# 修改openvswitch-switch配置文件
root@openstack:~# sed -i 's/# BRCOMPAT=no/BRCOMPAT=yes/g' /etc/default/openvswitch-switch

# 重启openvswitch-switch（注意ovs-brcompatd未启动）
root@openstack:~# /etc/init.d/openvswitch-switch restart
 * ovs-brcompatd is not running
 * Killing ovs-vswitchd (10699)
 * Killing ovsdb-server (10690)
FATAL: Error inserting brcompat (/lib/modules/3.5.0-28-generic/updates/dkms/brcompat.ko): Unknown symbol in module, or unknown parameter (see dmesg)
 * Inserting brcompat module
Module has probably not been built for this kernel.
Install the openvswitch-datapath-source package, then read
/usr/share/doc/openvswitch-datapath-source/README.Debian
FATAL: Error inserting brcompat (/lib/modules/3.5.0-28-generic/updates/dkms/brcompat.ko): Unknown symbol in module, or unknown parameter (see dmesg)
 * Inserting brcompat module

# 强制加载brcompat内核模块
root@openstack:~# /etc/init.d/openvswitch-switch force-reload-kmod
FATAL: Error inserting brcompat (/lib/modules/3.5.0-28-generic/updates/dkms/brcompat.ko): Unknown symbol in module, or unknown parameter (see dmesg)
 * Inserting brcompat module
Module has probably not been built for this kernel.
For instructions, read
/usr/share/doc/openvswitch-datapath-source/README.Debian
May 08 15:12:50|00001|stream_unix|ERR|/tmp/stream-unix.23796.0: connection to /var/run/openvswitch/db.sock failed: No such file or directory
May 08 15:12:50|00002|reconnect|WARN|unix:/var/run/openvswitch/db.sock: connection attempt failed (No such file or directory)
May 08 15:12:51|00003|stream_unix|ERR|/tmp/stream-unix.23796.1: connection to /var/run/openvswitch/db.sock failed: No such file or directory
May 08 15:12:51|00004|reconnect|WARN|unix:/var/run/openvswitch/db.sock: connection attempt failed (No such file or directory)
May 08 15:12:53|00005|stream_unix|ERR|/tmp/stream-unix.23796.2: connection to /var/run/openvswitch/db.sock failed: No such file or directory
May 08 15:12:53|00006|reconnect|WARN|unix:/var/run/openvswitch/db.sock: connection attempt failed (No such file or directory)
Alarm clock
 * Detected internal interfaces:
 * ovs-brcompatd is not running
 * ovs-vswitchd is not running
 * ovsdb-server is not running
 * Saving interface configuration
 * Removing openvswitch module
 * Inserting openvswitch module
 * Inserting brcompat module
 * Starting ovsdb-server
 * Configuring Open vSwitch system IDs
 * Starting ovs-vswitchd
 * Starting ovs-brcompatd
 * Restoring interface configuration
 * iptables already has a rule for gre, not explicitly enabling

# ovs-brcompatd、ovs-vswitchd、ovsdb-server均已启动
root@openstack:~# /etc/init.d/openvswitch-switch restart
 * Killing ovs-brcompatd (23823)
 * Killing ovs-vswitchd (23820)
 * Killing ovsdb-server (23811)
 * Starting ovsdb-server
 * Configuring Open vSwitch system IDs
 * Starting ovs-vswitchd
 * Starting ovs-brcompatd
 * iptables already has a rule for gre, not explicitly enabling

# 查看brcompat内核模块已挂载
root@openstack:~# lsmod | grep brcompat
brcompat               13513  0
openvswitch            84124  1 brcompat

# 如果还是有问题执行下面步骤，直到ovs-brcompatd、ovs-vswitchd、ovsdb-server都启动
root@openstack:~# apt-get install openvswitch-datapath-source
root@openstack:~# module-assistant auto-install openvswitch-datapath
root@openstack:~# /etc/init.d/openvswitch-switch force-reload-kmod
root@openstack:~# /etc/init.d/openvswitch-switch restart
文档参考：http://blog.scottlowe.org/2012/08/17/installing-kvm-and-open-vswitch-on-ubuntu/

root@openstack:~# ovs-vsctl add-port br-ex eth1
root@openstack:~# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
# This is an autoconfigured IPv6 interface
# iface eth0 inet6 auto
address 10.10.100.51
netmask 255.255.255.0

# For Exposing OpenStack API over the internet
auto eth1
iface eth1 inet manual
up ifconfig $IFACE 0.0.0.0 up
down ifconfig $IFACE down

auto br-ex
iface br-ex inet static
address 192.168.200.51
netmask 255.255.255.0
gateway 192.168.200.1
dns-nameservers 8.8.8.8

# 重启网络服务
root@openstack:~# /etc/init.d/networking restart

# 查看网络配置情况
root@openstack:~# ifconfig
br-ex     Link encap:Ethernet  HWaddr 00:1c:42:7b:0b:08
          inet addr:192.168.200.51  Bcast:192.168.200.255  Mask:255.255.255.0
          inet6 addr: fdb2:2c26:f4e4:0:ec31:e966:3021:cf8e/64 Scope:Global
          inet6 addr: fdb2:2c26:f4e4:0:21c:42ff:fe7b:b08/64 Scope:Global
          inet6 addr: fe80::21c:42ff:fe7b:b08/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:34186 errors:0 dropped:0 overruns:0 frame:0
          TX packets:21239 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:49979416 (49.9 MB)  TX bytes:1200632 (1.2 MB)

eth0      Link encap:Ethernet  HWaddr 00:1c:42:c7:d2:f5
          inet addr:10.10.100.51  Bcast:10.10.100.255  Mask:255.255.255.0
          inet6 addr: fdb2:2c26:f4e4:1:b990:97f:44f4:a7b5/64 Scope:Global
          inet6 addr: fdb2:2c26:f4e4:1:21c:42ff:fec7:d2f5/64 Scope:Global
          inet6 addr: fe80::21c:42ff:fec7:d2f5/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:10643 errors:0 dropped:0 overruns:0 frame:0
          TX packets:7711 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:867749 (867.7 KB)  TX bytes:1587488 (1.5 MB)

eth1      Link encap:Ethernet  HWaddr 00:1c:42:7b:0b:08
          inet6 addr: fe80::21c:42ff:fe7b:b08/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:116773 errors:0 dropped:0 overruns:0 frame:0
          TX packets:81107 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:161528642 (161.5 MB)  TX bytes:6142027 (6.1 MB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:278955 errors:0 dropped:0 overruns:0 frame:0
          TX packets:278955 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:48151158 (48.1 MB)  TX bytes:48151158 (48.1 MB)

root@openstack:~# ovs-vsctl add-br br-int
root@openstack:~# ovs-vsctl list-br
br-ex
br-int
root@openstack:~# ovs-vsctl show
b7e9e54f-d5d8-462e-bdf8-3565a4628cf3
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
    Bridge br-ex
        Port "eth1"
            Interface "eth1"
        Port br-ex
            Interface br-ex
                type: internal
    ovs_version: "1.4.0+build0"


root@openstack:~# apt-get install -y quantum-server quantum-plugin-openvswitch quantum-plugin-openvswitch-agent dnsmasq quantum-dhcp-agent quantum-l3-agent quantum-plugin-openvswitch-agent

root@openstack:/etc/init.d# vi /etc/quantum/l3_agent.ini
root@openstack:/etc/init.d# cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done
root@openstack:~# service dnsmasq restart

root@openstack:/etc/init.d# apt-get install -y kvm libvirt-bin pm-utils
root@openstack:~# vi /etc/libvirt/libvirtd.conf
root@openstack:~# vi /etc/init/libvirt-bin.conf
root@openstack:~# vi /etc/default/libvirt-bin
root@openstack:~# service libvirt-bin restart
libvirt-bin stop/waiting
libvirt-bin start/running, process 4014

root@openstack:~# apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor nova-compute-kvm

# 有用用于测试的宿主机是pd虚拟机，所以不能使用kvm全虚拟化，改用qemu虚拟化
root@openstack:~# apt-get install nova-compute-qemu

root@openstack:~# vi /etc/nova/api-paste.ini
root@openstack:~# vi /etc/nova/nova.conf
root@openstack:~# vi /etc/nova/nova-compute.conf
root@openstack:~# nova-manage db sync
root@openstack:~# cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done
root@openstack:/etc/init.d# nova-manage service list
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        openstack                            internal         enabled    :-)   None
nova-conductor   openstack                            internal         enabled    :-)   None
nova-consoleauth openstack                            internal         enabled    :-)   None
nova-scheduler   openstack                            internal         enabled    :-)   None
nova-compute     openstack                            nova             enabled    :-)   None

root@openstack:/etc/init.d# apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

root@openstack:/etc/init.d# cd
root@openstack:~# sed -i 's/false/true/g' /etc/default/iscsitarget
root@openstack:~# service iscsitarget start
root@openstack:~# service open-iscsi start

root@openstack:~# vi /etc/cinder/api-paste.ini
root@openstack:~# vi /etc/cinder/cinder.conf
root@openstack:~# cinder-manage db sync

root@openstack:~# cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done

root@openstack:~# apt-get install openstack-dashboard memcached


