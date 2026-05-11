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
