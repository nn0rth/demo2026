1. [Настройка ISP](#1-Настройка-ISP)
2. [Настройка BR-RTR](#2-Настройка-BR-RTR)
3. [Настройка HQ-RTR](#3-Настройка-HQ-RTR)
4. [Настройка HQ-SRV](#4-Настройка-HQ-SRV)
5. [Настройка BR-SRV](#5-Настройка-BR-SRV)
6. [Настройка HQ-CLI](#6-Настройка-HQ-CLI)
7. [Дополнительная информация](#7-Дополнительная-информация)
8. [2 Модуль](#8-2-Модуль)

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

![dop-info](/pictures/shema.png)
![dop-info](/pictures/ip-set.png)

## 8. 2 Модуль

Часть 1: Настройка Samba DC на BR-SRV
1.1 Установка необходимых пакетов

```

apt-get update && apt-get install -y task-samba-dc

```

1.2 Очистка предыдущей конфигурации Samba (если была)

```

rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol

```

1.3 Развёртывание домена

Запустите интерактивное развёртывание:

```

samba-tool domain provision
```
При запросе параметров:

```
    Realm: AU-TEAM.IRPO (подставится автоматически)
    Domain: AU-TEAM (подставится автоматически)
    Server Role: dc (нажмите Enter)
    DNS backend: SAMBA_INTERNAL (нажмите Enter)
    DNS forwarder IP address: 192.168.100.2 (IP HQ-SRV или другой DNS)
    Administrator password: введите пароль (минимум 7 символов, буквы верхнего/нижнего регистра, цифры)

```
1.4 Настройка служб

Включаем и добавляем в автозагрузку службу samba:

```

systemctl enable --now samba

```

Настройка Kerberos:

```

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

```
Перезагружаем службу samba:

```

systemctl restart samba

```

1.5 Настройка DNS на BR-SRV

Редактируем resolv.conf для интерфейса:

```
echo "search au-team.irpo" > /etc/net/ifaces/ens192/resolv.conf
echo "nameserver 127.0.0.1" >> /etc/net/ifaces/ens192/resolv.conf

```
Перезагружаем сеть:

```

systemctl restart network

```
1.6 Проверка работоспособности домена

Просмотр информации о домене:

```

samba-tool domain info 127.0.0.1

```

Проверка SMB-шар:

```

smbclient -L 127.0.0.1 -U administrator

```

Введите пароль администратора. Должны отобразиться шары sysvol и netlogon.

1.7 Проверка DNS

Установка утилиты host (если не установлена):

```

apt-get install -y bind-utils

```

Проверка DNS-записей:

```

host au-team.irpo
host -t SRV _kerberos._udp.au-team.irpo
host -t SRV _ldap._tcp.au-team.irpo
host br-srv.au-team.irpo

```
1.8 Проверка Kerberos

Получение билета (имя домена в ВЕРХНЕМ регистре):

```

kinit Administrator@AU-TEAM.IRPO

```

Просмотр полученного билета:

```

klist

```

Часть 2: Создание пользователей и группы
2.1 Создание группы hq

```

samba-tool group add hq

```

Проверка:

```

samba-tool group list

```

2.2 Создание пользователей и добавление в группу

```

for i in {1..5}; do
  samba-tool user add hquser$i P@ssw0rd
  samba-tool user setexpiry hquser$i --noexpiry
  samba-tool group addmembers "hq" hquser$i
done

```
Проверка членства в группе:

```

samba-tool group listmembers hq

```

Часть 3: Ввод HQ-CLI в домен
3.1 Настройка сети на HQ-CLI

Задаём статические параметры адресации с указанием DNS-сервера BR-SRV.

Через графический интерфейс (Настройки сети → Проводное подключение → IPv4):

```

    Метод IPv4: Вручную
    Адрес: 192.168.200.2
    Маска: 24
    Шлюз: 192.168.200.1
    DNS: 192.168.0.2 (IP адрес BR-SRV)

```
Проверка разрешения доменного имени:

```

host au-team.irpo

```

3.2 Установка пакетов для ввода в домен

```

apt-get update && apt-get install -y task-auth-ad-sssd

```

3.3 Ввод в домен через Центр Управления Системой

1. Откройте Центр управления системой
2. Перейдите в раздел Пользователи → Аутентификация
3. Выберите Active Directory
4. Введите:
```
    Домен: au-team.irpo
    Имя компьютера: hq-cli
    Администратор: administrator
    Пароль: (пароль администратора домена)
```
5. Нажмите Применить
После ввода в домен необходимо перезагрузить машину.

Часть 4: Настройка ограниченного sudo для группы hq
4.1 Установка libnss-role

```

apt-get install -y libnss-role

```

Проверка, что модуль включён:

```

control libnss-role

```

Ожидаемый вывод: enabled

4.2 Связывание доменной группы с локальной группой wheel

```

roleadd hq wheel

```

Проверка:

```

rolelst

```

Ожидаемый вывод должен содержать строку:

hq:wheel

4.3 Настройка sudoers

Редактируем файл /etc/sudoers:

visudo
или
nano /etc/sudoers

Добавляем алиас для разрешённых команд:
Cmnd_Alias      SHELLCMD = /bin/cat, /bin/grep, /usr/bin/id

Добавляем правило для группы wheel:
WHEEL_USERS ALL=(ALL:ALL) SHELLCMD

Часть 5: Проверка работы
5.1 Вход под доменным пользователем

На экране входа HQ-CLI нажмите "Нет в списке?":
Введите:
```

    Логин: hquser3 (или любой созданный пользователь)
    Пароль: P@ssw0rd
```
5.2 Проверка разрешённых команд

```

sudo id
sudo cat /etc/hosts
sudo grep '127.0.0.1' /etc/hosts

```

Результат: все команды выполняются успешно.
5.3 Проверка запрещённых команд
```
sudo su -
```
Результат: отказано в доступе

Возможные проблемы и решения:
1Ошибка при развёртывании домена
Убедитесь, что пароль соответствует требованиям сложности
Проверьте, что hostname настроен корректно
2HQ-CLI не видит домен
Проверьте, что DNS указывает на BR-SRV
Убедитесь в сетевой связности: ping 192.168.0.2
3Пользователь не может войти
Проверьте службу sssd: systemctl status sssd
Проверьте логи: journalctl -u sssd
4sudo не работает
Проверьте синтаксис sudoers: visudo -c
Убедитесь, что libnss-role включён: control libnss-role
