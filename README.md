---

## 1. ISP

```bash
hostnamectl set-hostname isp
exec bash

# WAN (магистраль)
mcedit /etc/net/ifaces/ens3/options
TYPE=eth
BOOTPROTO=dhcp

# HQ
mkdir -p /etc/net/ifaces/ens20
echo "172.16.1.1/28" > /etc/net/ifaces/ens20/ipv4address
mcedit /etc/net/ifaces/ens20/options
TYPE=eth
BOOTPROTO=static

# BR
mkdir -p /etc/net/ifaces/ens21
echo "172.16.2.1/28" > /etc/net/ifaces/ens21/ipv4address
mcedit /etc/net/ifaces/ens21/options
TYPE=eth
BOOTPROTO=static

# IP Forward
mcedit /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

systemctl restart network

apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens3 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens3 -j MASQUERADE

iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

timedatectl set-timezone Europe/Moscow

2. HQ-RTR
Bashenable
configure terminal

hostname HQ-RTR
ip domain-name au-team.irpo

username net_admin password P@ssw0rd role admin

interface isp
 ip address 172.16.1.2/28
 ip nat outside
exit

interface vl100
 ip address 10.10.100.1/27
 ip nat inside
exit

interface vl200
 ip address 10.10.200.1/27
 ip nat inside
exit

interface vl999
 ip address 10.10.30.1/29
 ip nat inside
exit

port ge1
 service-instance ge1/vl100
  encapsulation dot1q 100 exact
  rewrite pop 1
  connect ip interface vl100
 exit
 service-instance ge1/vl200
  encapsulation dot1q 200 exact
  rewrite pop 1
  connect ip interface vl200
 exit
 service-instance ge1/vl999
  encapsulation dot1q 999 exact
  rewrite pop 1
  connect ip interface vl999
 exit

interface tunnel.0
 description GRE-to-BR-RTR
 ip address 10.10.10.1/30
 ip tunnel 172.16.1.2 172.16.2.2 mode gre
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

router ospf 1
 ospf router-id 10.10.10.1
 passive-interface default
 no passive-interface tunnel.0
 network 10.10.100.0/27 area 0
 network 10.10.200.0/27 area 0
 network 10.10.30.0/29 area 0
 network 10.10.10.0/30 area 0
exit

ip nat source dynamic inside-to-outside interface isp overload
ip route 0.0.0.0/0 172.16.1.1

no dhcp-server 1
ip pool VLAN200 10.10.200.2-10.10.200.30

dhcp-server 1
 pool VLAN200 1
  mask 255.255.255.224
  gateway 10.10.200.1
  dns 10.10.100.2
  domain-name au-team.irpo
 interface vl200
  dhcp-server 1
 exit

ntp timezone utc+3
write memory

3. HQ-SRV (DNS/BIND9)
Bashhostnamectl set-hostname hq-srv.au-team.irpo

mkdir -p /etc/net/ifaces/ens3
echo "10.10.100.2/27" > /etc/net/ifaces/ens3/ipv4address
echo "default via 10.10.100.1" > /etc/net/ifaces/ens3/ipv4route

mcedit /etc/net/ifaces/ens3/options
TYPE=eth
BOOTPROTO=static

systemctl restart network

useradd sshuser -u 2026
echo 'sshuser:P@ssw0rd' | chpasswd
gpasswd -a sshuser wheel

mcedit /etc/sudoers
%wheel ALL=(ALL:ALL) NOPASSWD: ALL

mcedit /etc/openssh/sshd_config
Port 2026
MaxAuthTries 2
Banner /etc/openssh/banner
AllowUsers sshuser

echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd

apt-get update && apt-get install bind bind-utils -y

# Настройка BIND9 (зоны)
# ... (можно добавить отдельным файлом)

4. HQ-CLI
Bashhostnamectl set-hostname hq-cli.au-team.irpo

mkdir -p /etc/net/ifaces/ens3
mcedit /etc/net/ifaces/ens3/options
TYPE=eth
BOOTPROTO=dhcp

echo "search au-team.irpo" > /etc/resolv.conf
echo "nameserver 10.10.100.2" >> /etc/resolv.conf

systemctl restart network
timedatectl set-timezone Europe/Moscow

5. BR-RTR
Bashenable
configure terminal

hostname BR-RTR
ip domain-name au-team.irpo

username net_admin password P@ssw0rd role admin

interface isp
 ip address 172.16.2.2/28
 ip nat outside
exit

interface br
 ip address 10.20.20.1/28
 ip nat inside
exit

port te1
 service-instance te1/br
  encapsulation untagged
  connect ip interface br
 exit

interface tunnel.0
 description GRE-to-HQ-RTR
 ip address 10.10.10.2/30
 ip tunnel 172.16.2.2 172.16.1.2 mode gre
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

router ospf 1
 ospf router-id 10.10.10.2
 passive-interface default
 no passive-interface tunnel.0
 network 10.20.20.0/28 area 0
 network 10.10.10.0/30 area 0
exit

ip nat source dynamic inside-to-outside interface isp overload
ip route 0.0.0.0/0 172.16.2.1

ntp timezone utc+3
write memory

6. BR-SRV
Bashhostnamectl set-hostname br-srv.au-team.irpo

mkdir -p /etc/net/ifaces/ens3
echo "10.20.20.2/28" > /etc/net/ifaces/ens3/ipv4address
echo "default via 10.20.20.1" > /etc/net/ifaces/ens3/ipv4route

mcedit /etc/net/ifaces/ens3/options
TYPE=eth
BOOTPROTO=static

useradd sshuser -u 2026
echo 'sshuser:P@ssw0rd' | chpasswd
gpasswd -a sshuser wheel

echo "search au-team.irpo" > /etc/resolv.conf
echo "nameserver 10.10.100.2" >> /etc/resolv.conf

systemctl restart network
timedatectl set-timezone Europe/Moscow

----------------
НЕКИЙ МОДУЛЬ 2

1. Настройка контроллера домена Samba DC
Установка:

Bash

apt-get update
apt-get install -y task-samba-dc bind bind-utils
Provision домена:

Bash

samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass='P@ssw0rd' --server-role=dc --dns-backend=BIND9_DLZ
Проверка:

Bash

systemctl enable --now samba
systemctl status samba
host -t SRV _ldap._tcp.au-team.irpo
2. Настройка файлового хранилища (RAID0)
Создание RAID0:

Bash

mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0

mkdir /raid
mount /dev/md0 /raid
Автомонтирование:

Bash

blkid /dev/md0
nano /etc/fstab
Добавить в fstab:

text

/dev/md0 /raid ext4 defaults 0 0
Проверка:

Bash

cat /proc/mdstat
df -h
3. Настройка NFS
Установка:

Bash

apt-get install -y nfs-kernel-server
mkdir -p /raid/nfs
chmod 777 /raid/nfs
Экспорт:

Bash

nano /etc/exports
Добавить:

text

/raid/nfs 192.168.0.0/16(rw,sync,no_subtree_check)
Применение:

Bash

exportfs -ra
systemctl restart nfs-server
showmount -e localhost
4. Настройка Chrony
Установка:

Bash

apt-get install -y chrony
Конфигурация:

Bash

nano /etc/chrony.conf
Обязательно добавить:

text

local stratum 5
allow 192.168.0.0/16
Перезапуск и проверка:

Bash

systemctl restart chronyd
chronyc sources
chronyc tracking
5. Настройка Ansible
Установка:

Bash

apt-get install -y ansible
Inventory:

Bash

nano /etc/ansible/hosts
Корректно по стенду:

ini

[hq]
hq-srv ansible_host=192.168.100.2
hq-cli ansible_host=192.168.100.3
hq-rtr ansible_host=192.168.100.1

[branch]
br-rtr ansible_host=192.168.200.1
Проверка:

Bash

ansible all -m ping
6. Настройка веб-приложения через Docker
Установка Docker:

Bash

apt-get install -y docker docker-compose
systemctl enable --now docker
Compose:

Bash

mkdir /opt/testapp
nano /opt/testapp/docker-compose.yml
Содержимое:

YAML

version: '3'
services:
  db:
    image: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: appdb
  app:
    image: testapp
    restart: always
    ports:
      - "8080:80"
Запуск:

Bash

docker-compose up -d
Проверка:

Bash

docker ps
7. Настройка веб-приложения на сервере
Установка nginx:

Bash

apt-get install -y nginx
systemctl enable --now nginx
Проверка:

Bash

systemctl status nginx
8. Настройка трансляции портов (Static NAT)
На маршрутизаторе:

text

enable
configure terminal

ip nat inside source static tcp 192.168.100.2 80 <WAN_IP> 80
ip nat inside source static tcp 192.168.100.2 443 <WAN_IP> 443
Назначение интерфейсов:

text

interface G0/0
 ip nat inside

interface G0/1
 ip nat outside
9. Настройка обратного прокси-сервера
Конфиг nginx:

Bash

nano /etc/nginx/sites-available/reverse-proxy
Содержимое полностью как в методичке:

nginx

server {
 listen 80;
 server_name web.au-team.irpo;
 location / {
 proxy_pass http://192.168.100.2:8080;
 }
}

server {
 listen 80;
 server_name docker.au-team.irpo;
}

location / {
 proxy_pass http://127.0.0.1:8080;
}
Активация:

Bash

ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
10. Настройка web-based аутентификации
Требуется:

Авторизация пользователей через веб-интерфейс
Интеграция с доменом
Проверка доступа через браузер
11. Установка Яндекс.Браузера
Установка:

Bash

apt-get install -y yandex-browser-stable
Проверка:

Bash

yandex-browser
Итоговая проверка
Проверить по порядку перед сдачей:

Bash

systemctl status samba
cat /proc/mdstat
showmount -e
chronyc sources
ansible all -m ping
docker ps
nginx -t

