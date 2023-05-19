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
ip tunnel add tun1 mode gre local [локальный IP] remote [удаленный IP]
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
	}
}
:wq!
nft -f /etc/nftables.conf

#Проверка порта
ssh -l user [ip] -p 2244 //На правом 2222 


