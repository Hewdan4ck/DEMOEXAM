#Настройка IP на Debian
vim /etc/network/interfaces

auto [Интерфейс]
iface [Интерфейс] inet static
address [ip-адрес]/[Префикс]
gateway [ip-адрес шлюза]

#Настройка IP на CentOS
nmtui

#Настройка hostname
echo [Hostname] > /etc/hostname

#
