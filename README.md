# Полная исправленная конфигурация по методике СиСА 2026

**Автор:** prograber  
**Дата:** Май 2026

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

# Настройка BIND9...
# (полный текст зоны можно вставить при необходимости)
