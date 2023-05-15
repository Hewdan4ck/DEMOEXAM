#Настройка IP на Debian
vim /etc/network/interfaces

auto [Интерфейс]
iface [Интерфейс] inet static
address [ip-адрес]
gateway [ip-адрес шлюза]

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
