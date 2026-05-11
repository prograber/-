 ==================================================
 1. ISP
 hostnamectl set-hostname isp
 exec bash
 # WAN (магистраль)
 mcedit /etc/net/ifaces/ens3/options
 TYPE=eth
 BOOTPROTO=dhcp
 # HQ
 mkdir-p /etc/net/ifaces/ens20
 echo "172.16.1.1/28" > /etc/net/ifaces/ens20/ipv4address
 mcedit /etc/net/ifaces/ens20/options
 TYPE=eth
 BOOTPROTO=static
 # BR
 mkdir-p /etc/net/ifaces/ens21
 echo "172.16.2.1/28" > /etc/net/ifaces/ens21/ipv4address
 mcedit /etc/net/ifaces/ens21/options
 TYPE=eth
 BOOTPROTO=static
 # IP Forward
 mcedit /etc/net/sysctl.conf
 net.ipv4.ip_forward = 1
 systemctl restart network
 apt-get update && apt-get install iptables-y
 iptables-t nat-A POSTROUTING-s 172.16.1.0/28-o ens3-j MASQUERADE
 iptables-t nat-A POSTROUTING-s 172.16.2.0/28-o ens3-j MASQUERADE
 iptables-save > /etc/sysconfig/iptables
 systemctl enable--now iptables
 timedatectl set-timezone Europe/Moscow
 1
2. HQ-RTR
 enable
 configure terminal
 hostname HQ-RTR
 ip domain-name au-team.irpo
 username net_admin
 password P@ssw0rd
 role admin
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
 2
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
 hostnamectl set-hostname hq-srv.au-team.irpo
 mkdir-p /etc/net/ifaces/ens3
 echo "10.10.100.2/27" > /etc/net/ifaces/ens3/ipv4address
 echo "default via 10.10.100.1" > /etc/net/ifaces/ens3/ipv4route
 mcedit /etc/net/ifaces/ens3/options
 TYPE=eth
 BOOTPROTO=static
 3
systemctl restart network
 useradd sshuser-u 2026
 echo 'sshuser:P@ssw0rd' | chpasswd
 gpasswd-a sshuser wheel
 mcedit /etc/sudoers
 %wheel ALL=(ALL:ALL) NOPASSWD: ALL
 mcedit /etc/openssh/sshd_config
 Port 2026
 MaxAuthTries 2
 Banner /etc/openssh/banner
 AllowUsers sshuser
 echo "Authorized access only" > /etc/openssh/banner
 systemctl restart sshd
 apt-get update && apt-get install bind bind-utils-y
 mcedit /var/lib/bind/etc/options.conf
 listen-on port 53 { 10.10.100.2; any; };
 forwarders { 77.88.8.8; };
 allow-query { any; };
 mcedit /var/lib/bind/etc/rfc1912.conf
 zone "au-team.irpo" {
 type master;
 file "au-team.irpo";
 };
 zone "100.10.10.in-addr.arpa" {
 type master;
 file "10.10.100.in-addr.arpa";
 };
 zone "200.10.10.in-addr.arpa" {
 type master;
 file "10.10.200.in-addr.arpa";
 };
 cd /var/lib/bind/etc/zone
 cp empty au-team.irpo
 cp empty 10.10.100.in-addr.arpa
 cp empty 10.10.200.in-addr.arpa
 # ПРЯМАЯ ЗОНА
 mcedit au-team.irpo
 $TTL 1D
 @ IN SOA hq-srv.au-team.irpo. root.au-team.irpo. (
 2026051001 12H 1H 1W 1H )
 @ IN NS hq-srv.au-team.irpo.
 4
hq-rtr IN A
 br-rtr IN A
 hq-srv IN A
 hq-cli IN A
 br-srv IN A
 docker IN A
 web
 IN A
 10.10.100.1
 10.20.20.1
 10.10.100.2
 10.10.200.2
 10.20.20.2
 172.16.1.1
 172.16.2.1
 # ОБРАТНАЯ VLAN100
 mcedit 10.10.100.in-addr.arpa
 1 IN PTR hq-rtr.au-team.irpo.
 2 IN PTR hq-srv.au-team.irpo.
 # ОБРАТНАЯ VLAN200
 mcedit 10.10.200.in-addr.arpa
 1 IN PTR hq-rtr.au-team.irpo.
 2 IN PTR hq-cli.au-team.irpo.
 rndc-confgen > /etc/bind/rndc.key
 sed-i '6,$d' /etc/bind/rndc.key
 chown-R root:named /var/lib/bind/etc/zone/*
 named-checkconf
 named-checkconf-z
 systemctl enable--now bind
 echo "search au-team.irpo" > /etc/resolv.conf
 echo "nameserver 10.10.100.2" >> /etc/resolv.conf
 timedatectl set-timezone Europe/Moscow
 4. HQ-CLI
 hostnamectl set-hostname hq-cli.au-team.irpo
 mkdir-p /etc/net/ifaces/ens3
 mcedit /etc/net/ifaces/ens3/options
 TYPE=eth
 BOOTPROTO=dhcp
 echo "search au-team.irpo" > /etc/resolv.conf
 echo "nameserver 10.10.100.2" >> /etc/resolv.conf
 5
systemctl restart network
 timedatectl set-timezone Europe/Moscow
 5. BR-RTR
 enable
 configure terminal
 hostname BR-RTR
 ip domain-name au-team.irpo
 username net_admin
 password P@ssw0rd
 role admin
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
 6
ip route 0.0.0.0/0 172.16.2.1
 ntp timezone utc+3
 write memory
 6. BR-SRV
 hostnamectl set-hostname br-srv.au-team.irpo
 mkdir-p /etc/net/ifaces/ens3
 echo "10.20.20.2/28" > /etc/net/ifaces/ens3/ipv4address
 echo "default via 10.20.20.1" > /etc/net/ifaces/ens3/ipv4route
 mcedit /etc/net/ifaces/ens3/options
 TYPE=eth
 BOOTPROTO=static
 useradd sshuser-u 2026
 echo 'sshuser:P@ssw0rd' | chpasswd
 gpasswd-a sshuser wheel
 echo "search au-team.irpo" > /etc/resolv.conf
 echo "nameserver 10.10.100.2" >> /etc/resolv.conf
 systemctl restart network
 timedatectl set-timezone Europe/Moscow
 7
