#Настройка IP на Debian
vim /etc/network/interfaces

auto [Интерфейс]
iface [Интерфейс] inet static
address [ip-адрес]
gateway [ip-адрес шлюза]
:wq

#Перенаправления пакетов
vim /etc/sysctl.conf
Нужно раскомментировать или дописать
net.ipv4.ip_forward=1

echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
sysctl -p


#Настройка IP на CentOS
nmtui

#Настройка hostname
echo [Hostname] > /etc/hostname

#NAT
vim /etc/nftables.conf
table ip nat {
	chain postrouting {
		type nat hook postrouting priority 0; policy accept;
		ip saddr [Диапазон натируемых адресов] oif "[Выходной интерфейс]" masquerade;
	}
}
:wq
systemctl enable --now nftables

#NAT для ISP, если понадобится устанавливать пакеты с инета
vim /etc/nftables.conf
table ip nat {
	chain postrouting {
		type nat hook postrouting priority 0; policy accept;
		ip saddr 0.0.0.0/0 oif "[Выходной интерфейс]" masquerade;
	}
}
:wq
systemctl enable --now nftables

#Nameserver для установки пакетов с инета
echo nameserver 8.8.8.8 > /etc/resolv.conf

#Установка репозитория для дебиана
echo deb http://mirror.yandex.ru/debian bullseye main contrib >> /etc/apt/sources.list
apt update

#Применение и проверка правил nftables 
nft -f /etc/nftables.conf
nft list ruleset

#GRE на RTR
vim /etc/gre.up
#!/bin/bash
ip tunnel add tun1 mode gre local [локальный IP] remote [удаленный IP] ttl 64
ip addr add [виртуальный ip локального роутера (10.5.5.1)]/30 dev tun1
ip link set tun1 up
ip route add [удаленная сеть]/[префикс] via [виртуальный ip удаленного роутера (например, 10.5.5.2)]
:wq

chmod +x /etc/gre.up

vim /etc/crontab
@reboot		root	/etc/gre.up
:wq

#IPSEC Strongswan на RTR
apt install strongswan -y

vim /etc/ipsec.conf
conn vpn
	auto=start
	type=tunnel
	authby=secret
	left=[локальный ip]
	right=[удаленный ip]
	leftsubnet=0.0.0.0/0
	rightsubnet=0.0.0.0/0
	leftprotoport=gre
	rightprotoport=gre
	ike=aes128-sha256-modp3072
	esp=aes128-sha256
:wq

vim /etc/ipsec.secrets
[локальный ip] [удаленный ip] : PSK "P@ssw0rd"

#Настройка фильтрации VPN на RTR
vim /etc/nftables.conf
Дописываем
table inet filter {
	chain input {
		type filter hook input priority 0;
		udp dport 53 accept; //На правом роутере не требуется
		tcp dport 80 accept;
		tcp dport 443 accept;
		ct state {established,related} accept;
		ip protocol gre accept;
		ip protocol icmp accept;
		ip protocol ospf accept;
		udp dport 500 accept;
		ip saddr [локальная подсеть] accept;
		ip saddr [удаленная подсеть] accept;
		tcp dport 2244 accept; //На правом роутере 2222
		ip version 4 drop;
	}
	chain forward {
		type filter hook forward priority 0;
	}
	chain output {
		type filter hoot output priority 0;
	}
}
:wq
nft -f /etc/nftables.conf

#Проброс портов
vim /etc/nftables.conf
Дописываем
table ip nat {
	chain postrouting {
		...
	}
	chain prerouting {
		type nat hook prerouting priority 0; policy accept;
		tcp dport 2244 dnat to [локальный ip устройства для проброса]:22; //На правом проброс на 2222
		iifname "[Внешний интерфейс]" udp dport 53 dnat to 192.168.200.200:53; //Только на левом роутере
	}
}
:wq!
nft -f /etc/nftables.conf

#Проверка порта
ssh -l user [ip] -p 2244 //На правом 2222 

#Динамическая маршрутизация
vim /etc/gre.up
#ip route add 192.168.200.0/24 via 10.5.5.1
:wq
reboot
apt install frr -y
vim /etc/frr/daemons
ospdfd=yes
:wq
systemctl restart frr
vtysh

RTR-R# conf t
RTR-R(config)# router ospf
RTR-R(config-router)# network [сеть gre]/30 area 0
RTR-R(config-router)# network [локальная сеть]/24 area 0
RTR-R(config-router)# end
RTR-R# show ip ospf nei
RTR-R# show ip ospf route
RTR-R# wr
RTR-R# exit

#DNS ISP
apt install bind9 -y
vim /etc/bind/named.conf.options
options {
	directory "/var/cache/bind";
	dnssec-validation auto;	
	allow-query { any; };
	recursion yes;
	listen-on { any; };
};
:wq
vim /etc/bind/named.conf.default-zones
zone "." {
	type hint;
	file "/usr/share/dns/root.hints";
};

zone "demo.wsr" {
	type master;
	file "/etc/bind/demo.wsr";
	forwarders {};
};
:wq
cp /etc/bind/db.local /etc/bind/demo.wsr
vim /etc/bind/demo.wsr
$TTL 604800
@	IN	SOA	demo.wsr. root.demo.wsr. (
				2
				604800
				86400
				2419200
				604800 )
@	IN	NS	demo.wsr.
@	IN	A	3.3.3.1
isp	IN	A	3.3.3.1
www	IN	A	4.4.4.100
www	IN	A	5.5.5.100
internet	IN	CNAME	isp.demo.wsr.

$ORIGIN	int.demo.wsr.
@	IN	NS	int.demo.wsr.
@	IN	A	4.4.4.100
:wq
named-checkconf		//Проверка конфиги
named-checkconf -z 	//Проверка зон
systemctl restart bind9
host demo.wsr 3.3.3.1	//На роутерах	
host int.demo.wsr 3.3.3.1

#Проверка DNS windows
host www.demo.wsr 3.3.3.1
host srv.int.demo.wsr 4.4.4.100
host ntp.int.demo.wsr 4.4.4.100

#NTP ISP
apt install chrony -y
vim /etc/chrony/chrony.conf
confdir /etc/chrony/conf.d
server 127.0.0.1
allow 5.5.5.100/32
allow 4.4.4.100/32
allow 3.3.3.10/32
local stratum 3
...
:wq
systemctl restart chrony
chronyc sources
chronyc clients

#NTP RTR
apt install chrony -y
vim /etc/chrony/chrony.conf
confdir /etc/chrony/conf.d
server 192.168.200.200 iburst //IP SRV
...
:wq
systemctl restart chrony
chronyc sources

#SMB client WEB-L и WEB-R
apt install cifs-utils -y
vim /etc/fstab
...
//192.168.200.200/shares /opt/share cifs defaults,credentials=/etc/pass,_netdev 0 0
mkdir /opt/share
echo "username=Administrator
password=P@ssw0rd" > /etc/pass
mount -a
df -h

#Nameserver на всех устройствах, кроме ISP и Windows клиентах
echo nameserver 192.168.200.200 > /etc/resolv.conf //IP SRV

#Docker WEB-L и WEB-R 
//Для начала нужно добавить второй CD-ROM с диском docker.iso

mount /dev/sr1 /media/cdrom
cd /media/cdrom
cp appdockerdemo.tar.gz /root/
cd
tar -xvf appdockerdemo.tar.gz
apt install docker docker.io -y
docker image load -i appdocker0.zip
//docker run -d appdocker0:latest
//docker kill [Имя докера в конце docker ps]
docker run -d -p 80:5000 appdocker0	//80 порт внешний, 5000 порт внутренний
docker image ls
docker ps
