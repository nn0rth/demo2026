1. [Настройка ISP](#1-Настройка-ISP)
2. [Настройка BR-RTR](#2-Настройка-BR-RTR)
3. [Настройка HQ-RTR](#3-Настройка-HQ-RTR)
4. [Настройка HQ-SRV](#4-Настройка-HQ-SRV)
5. [Настройка BR-SRV](#5-Настройка-BR-SRV)
6. [Настройка HQ-CLI](#6-Настройка-HQ-CLI)
7. [Дополнительная информация](#7-Дополнительная-информация)

## 1. Настройкка ISP

=========ISP=========

```
root/toor

```
-------имя хоста---------

```
hostnamectl hostname ISP
exec bash

```
-------настройка ВНЕШНЕГО интерфейса---------

```
cat /etc/net/ifaces/enp7s1/options

```
-------настройка ВНУТРЕННИХ интерфейсов---------

```
mkdir -p /etc/net/ifaces/enp7s{2,3}

echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{2,3}/options

echo '172.16.1.1/28' > /etc/net/ifaces/enp7s2/ipv4address
echo '172.16.2.1/28' > /etc/net/ifaces/enp7s3/ipv4address

```
-------настройка NAT---------

```
apt-get update && apt-get install nftables -y

```
-------------nftables.nft-------------

```
cat << EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
 chain postrouting {
 type nat hook postrouting priority srcnat;
 oifname "enp7s1" masquerade
 }
}
EOF

cat /etc/nftables/nftables.nft

systemctl enable --now nftables

```
-------включение маршрутизации---------

```
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
systemctl restart network
sysctl net.ipv4.ip_forward

```

## 2. Настройка BR-RTR

=======BR-RTR=======

```

root/toor

```

------имя хоста---------

```

hostnamectl hostname br-rtr.au-team.irpo
exec bash

```

-------NETWORK-----------

```

mkdir -p /etc/net/ifaces/{enp7s{1,2},gre1}
echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{1,2}/options

```

-------to ISP-------

```

echo '172.16.2.2/28' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 172.16.2.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

```

-------to BR-SRV---

```

echo '192.168.1.1/28' > /etc/net/ifaces/enp7s2/ipv4address

```

-------включение маршрутизации---------

```

sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf

```

-----------GRE----------

```

cat << EOF > /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.2.2
TUNREMOTE=172.16.1.2
TUNTTL=64
TUNOPTIONS='ttl 64'
EOF

cat /etc/net/ifaces/gre1/options

echo "10.10.10.2/30" > /etc/net/ifaces/gre1/ipv4address

systemctl restart network
ip -br -c a

```

-------установка необходимого ПО-------

```

apt-get update && apt-get install sudo tzdata frr nftables -y

```

-------смена DNS-------------

```

rm -f /etc/net/ifaces/enp7s1/resolv.conf
echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/enp7s2/resolv.conf

```

---------nftables-----------

```

cat << EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
chain postrouting {
type nat hook postrouting priority srcnat;
oifname "enp7s1" masquerade
}
}
EOF

```

---------TIMEZONE-----------

```

timedatectl set-timezone Europe/Moscow

```

--------настройка net_admin-------------

```
useradd net_admin
echo "net_admin:P@ssw0rd" | chpasswd
usermod -aG wheel net_admin
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
su -l net_admin
sudo id

```

--------OSPF-----------------

```

sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons


cat << 'EOF' > /etc/frr/frr.conf

interface gre1
ip ospf area 0
ip ospf authentication
ip ospf authentication-key P@ssw0rd
no ip ospf passive
exit
!
interface enp7s2
ip ospf area 0
exit
!
router ospf
passive-interface default
exit
EOF


systemctl restart network
systemctl enable --now nftables frr
cat /etc/resolv.conf
ip r

```

## 3. Настройка HQ-RTR

=======HQ-RTR=======

```

root/toor

```

-------имя хоста---------

```

hostnamectl hostname hq-rtr.au-team.irpo
exec bash

```

---------------NETWORK--------------------

```

mkdir -p /etc/net/ifaces/{enp7s{1,2},vlan{100,200,999},gre1}
echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{1,2}/options

```

-------to ISP-------

```

echo '172.16.1.2/28' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 172.16.1.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

```

-------настройка VLAN-------------

```

echo $'100\n200\n999' | xargs -i bash -c 'echo -e "TYPE=vlan\nHOST=enp7s2\nVID={}" > /etc/net/ifaces/vlan{}/options'

cat /etc/net/ifaces/vlan999/options

echo '192.168.100.1/27' > /etc/net/ifaces/vlan100/ipv4address
echo '192.168.200.1/28' > /etc/net/ifaces/vlan200/ipv4address
echo '192.168.99.1/29' > /etc/net/ifaces/vlan999/ipv4address

```

-------включение маршрутизации---------

```

sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf

```
------------GRE-----------------

```

cat << EOF > /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.1.2
TUNREMOTE=172.16.2.2
TUNOPTIONS='ttl 64'
EOF


cat /etc/net/ifaces/gre1/options

echo "10.10.10.1/30" > /etc/net/ifaces/gre1/ipv4address

systemctl restart network

ip -br -c a
ping 10.10.10.2 -c 3
ping zz.ru -c 2

```

-------установка необходимого ПО-------

```

apt-get update && apt-get install sudo tzdata frr dnsmasq nftables -y

```

-------смена DNS-------------

```

rm -f /etc/net/ifaces/enp7s1/resolv.conf
echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/vlan100/resolv.conf

```

----------nftables------------

```

cat << EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
chain postrouting {
type nat hook postrouting priority srcnat;
oifname "enp7s1" masquerade
}
}
EOF


systemctl enable --now nftables

```
---------TIMEZONE-----------

```

timedatectl set-timezone Europe/Moscow

```

настройка net_admin-------------

```

useradd net_admin
echo "net_admin:P@ssw0rd" | chpasswd
usermod -aG wheel net_admin
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
su -l net_admin
sudo id

```

-----------OSPF-----------------

```

sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons


cat << 'EOF' > /etc/frr/frr.conf
interface gre
no ip ospf passive
exit
!
interface gre1
ip ospf area 0
ip ospf authentication
ip ospf authentication-key P@ssw0rd
no ip ospf passive
exit
!
interface vlan100
ip ospf area 0
exit
!
interface vlan200
ip ospf area 0
exit
!
interface vlan999
ip ospf area 0
exit
!
router ospf
passive-interface default
exit

EOF

```

--------------DHCP-----------------

```

sed -i 's/AUTO_LOCAL_RESOLVER=yes/AUTO_LOCAL_RESOLVER=no/' /etc/sysconfig/dnsmasq; grep AUTO_LOCAL_RESOLVER /etc/sysconfig/dnsmasq


cat << 'EOF' > /etc/dnsmasq.conf
port=0
interface=vlan200
listen-address=192.168.200.1
dhcp-authoritative
dhcp-range=interface:vlan200,192.168.200.2,192.168.200.2,255.255.255.240,6h
dhcp-option=3,192.168.200.1
dhcp-option=6,192.168.100.2
leasefile-ro
EOF


systemctl enable --now frr dnsmasq ; ss -lun | grep 67

systemctl restart network
cat /etc/resolv.conf
ip r | grep ospf

```

## 4. Настройка HQ-SRV

=======HQ-SRV=======

```

root/toor

```

СТАВИМ ТЕГ VLAN 100 НА ИНТЕРФЕЙС

```

mkdir -p /etc/net/ifaces/vlan200
echo -e "TYPE=vlan\nHOST=enp7s1\nVID=200" > /etc/net/ifaces/vlan200/options
systemctl restart network
ip -br -c a

```

------имя хоста/NTP---------

```

hostnamectl hostname HQ-SRV.au-team.irpo
exec bash
timedatectl set-timezone Europe/Moscow

```

<-------настройка интерфейсов-------------

```

echo 'TYPE=eth' > /etc/net/ifaces/enp7s1/options
echo '192.168.100.2/27' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 192.168.100.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
ping zz.ru -c3

```

-------настройка sshuser-----------------

```

useradd -u 2026 sshuser
echo "sshuser:P@ssw0rd" | chpasswd
usermod -aG wheel sshuser
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
su -l sshuser
sudo id

```

-------настройка SSH---------------------

```

echo "Authorized access only" > /etc/openssh/banner
echo -e "Port 2026\nMaxAuthTries 2\nAllowUsers sshuser\nBanner /etc/openssh/banner\n" >> /etc/openssh/sshd_config
systemctl restart sshd
ss -ltnp | grep sshd

ssh sshuser@127.0.0.1 -p 2026

```

-------установка необходимого ПО-------

```

apt-get update && apt-get install bind bind-utils -y

```
-------смена DNS-------------

```

echo $'search au-team.irpo\nnameserver 127.0.0.1' > /etc/net/ifaces/enp7s1/resolv.conf

```

-------BIND9-----------------

```

rndc-confgen -a -c /etc/bind/rndc.key


cat << 'EOF' > /etc/bind/options.conf
logging { };
options {
listen-on { localnets; 127.0.0.1; };
forwarders { 77.88.8.7; 77.88.8.3; };
recursion yes;
allow-recursion { any; };
allow-query { any; };
dnssec-validation no;

directory "/etc/bind/zone";
dump-file "/var/run/named/named_dump.db";
statistics-file "/var/run/named/named.stats";
recursing-file "/var/run/named/named.recursing";
secroots-file "/var/run/named/named.secroots";
pid-file none;
};
zone "au-team.irpo" {
type master;
file "au-team.irpo";
};
zone "168.192.in-addr.arpa" {
type master;
file "168.192.in-addr.arpa";
};
EOF



cat << 'EOF' > /etc/bind/zone/au-team.irpo
$TTL 1D
@ IN SOA au-team.irpo. root.au-team.irpo. (
2025020600 ; serial
12H        ; refresh
1H         ; retry
1W         ; expire
1H         ; ncache
)
@      IN NS  hq-srv.au-team.irpo.
hq-rtr IN A   192.168.100.1
hq-srv IN A   192.168.100.2
hq-cli IN A   192.168.200.2
br-rtr IN A   192.168.1.1
br-srv IN A   192.168.1.2
docker IN A   172.16.1.1
web    IN A   172.16.2.1
EOF



cat << 'EOF' > /etc/bind/zone/168.192.in-addr.arpa
$TTL 1D
@ IN SOA au-team.irpo. root.au-team.irpo. (
2025020600 ; serial
12H        ; refresh
1H         ; retry
1W         ; expire
1H         ; ncache
)
IN NS  au-team.irpo.
1.100  IN PTR hq-rtr.au-team.irpo.
2.100  IN PTR hq-srv.au-team.irpo.
2.200  IN PTR hq-cli.au-team.irpo.
EOF


chown :named /etc/bind/zone/au-team.irpo /etc/bind/zone/168.192.in-addr.arpa
systemctl enable --now bind
service network restart
host br-rtr
host -t PTR 192.168.100.2

```

## 5. Настройка BR-SRV

=======BR-SRV=======

```

root/toor

```

-------имя хоста/NTP---------

```

hostnamectl hostname BR-SRV.au-team.irpo
exec bash
timedatectl set-timezone Europe/Moscow

```

-------настройка интерфейсов-------------

```

echo 'TYPE=eth' > /etc/net/ifaces/enp7s1/options
echo '192.168.1.2/26' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 192.168.1.1' > /etc/net/ifaces/enp7s1/ipv4route
echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
ping hq-srv -c 3

```

-------настройка sshuser-----------------

```

useradd -u 2026 sshuser
echo "sshuser:P@ssw0rd" | chpasswd
usermod -aG wheel sshuser
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
su -l sshuser
sudo id

```

-------настройка SSH---------------------

```
echo "Authorized access only" > /etc/openssh/banner
echo -e "Port 2026\nMaxAuthTries 2\nAllowUsers sshuser\nBanner /etc/openssh/banner\n" >> /etc/openssh/sshd_config
systemctl restart sshd
ss -ltnp | grep sshd

```

## 6. Настройка HQ-CLI

=======HQ-CLI=======

СТАВИМ ТЕГ VLAN 200 НА ИНТЕРФЕЙС

```

mkdir -p /etc/net/ifaces/vlan200
echo -e "TYPE=vlan\nHOST=enp7s1\nVID=200" > /etc/net/ifaces/vlan200/options
systemctl restart network
ip -br -c a

```

-------имя хоста/NTP---------

```

hostnamectl hostname HQ-CLI.au-team.irpo
exec bash
timedatectl set-timezone Europe/Moscow
ip -br -c a

```

## 7. Дополнительная информация

![dop-info](../pictures/shema.png)



![dop-info](../pictures/ip-set.jpg)
