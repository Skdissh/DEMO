# DEMO
1. Изменение наименований машин
hostnamectl установить имя хоста isp.au-team.irpo;
выполнить bash
2. Создание связей между машинами
Провайдер (nmtui)
isp-hq 172.16.4.1/28
isp-br 172.16.5.1/28
HQ-RTR (nmtui)
isp — hq-rtr — 172.16.4.2/28 шлюз 172.16.4.1
hq — в 192.168.10.1/26
BR-RTR (nmtui)
isp — br-rtr — 172.16.5.2/28 шлюз 172.16.5.1
br — в 192.168.60.1/27
HQ-SRV (Графика)
IP-адрес: 192.168.10.2 Маска: 26 Шлюз: 192.168.10.1
HQ-CLI (Графика)
IP-адрес: 192.168.10.3 Маска: 28 Шлюз: 192.168.10.1
BR-SRV (Графика)
IP-адрес: 192.168.60.2 Маска: 27 Шлюз: 192.168.60.1
BR-DC (Графика)
IP-адрес: 192.168.60.3 Маска: 27 (255.255.255.224) Шлюз: 192.168.60.1
3. Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV
Создание пользователей
useradd -m -u 1010 sshuser
passwd sshuser
# P@ssw0rd
nano /etc/sudoers
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
3.1 Создайте пользователя net_admin на серверах HQ-SRV и BR-SRV
добавляемый пользователь net_admin
passwd net_admin
# P@$$word
nano /etc/sudoers
net_admin ALL=(ALL:ALL)NOPASSWD:ALL
4. Настройка безопасного удалённого доступа на серверах HQ-SRV и BR-SRV
nano /etc/mybanner
Authorized access only
nano /etc/openssh/sshd_config
port 2024
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser
systemctl restart sshd.service
5. Настроить IP-туннель HQ и BR
Провайдер
nano /etc/net/sysctl.conf
net ipv4 forwarding = 1
BR-RTR
Имя tun1
режим: GRE
Локальный: 172.16.5.2
Удаленный: 172.16.4.2
Адрес: 192.168.0.2/24
Шлюз: 192.168.0.1
ШТАБ-квартира-RTR
Имя tun1
режим: GRE
Локальный: 172.16.4.2
Удаленный: 172.16.5.2
Адрес: 192.168.0.1/24
Шлюз: 192.168.0.2
#Перезагрузить BR-RTR и HQ-RTR и не забыть перезапустить интернет
6. Настройка протокола динамической конфигурации хостов HQ-RTR
nano /etc/sysconfig/dhcpd
DHCPARGS=ens35
cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
option domain-name "au-team.irpo";
option domain-name-servers 192.168.10.2;

default-lease-time 6000;
max-lease-time 72000;

authoritative;
subnet 192.168.10.0 netmask 255.255.255.192 {
    range 192.168.10.10 192.168.10.20;
    option routers 192.168.10.1;
}

systemctl enable –-now dhcpd
7. Настройка OSPF HQ-RTR и BR-RTR
#Отключить FireWall systemctl disable firewalld --now hq-r
nano /etc/frr/daemons
ospfd=yes
ШТАБ-квартира-RTR
systemctl включить --сейчас frr
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.10.1/26 area 0 
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
BR-RTR
vtysh
conf t
маршрутизатор ospf
пассивный интерфейс по умолчанию
сеть 192.168.0.0/24 область 0
сеть 192.168.60.1/27 область 0
выход
интерфейс tun1
нет трансляции IP-адресов в сети ospf
нет пассивного IP-адреса в сети ospf
выход
сделать wr
выход
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
# Перезагрузить BR-RTR и HQ-RTR
Проверка работы OSPF
вытыш
show ip ospf neighbor
8. Настройка DNS для офисов HQ-SRV и BR-SRV (если успею!)
ШТАБ-квартира-SRV
nano /etc/bind/options.conf
listen on { any; };
forward first;
forwarders { 77.88.8.8; };
systemctl enable bind —now
nano /etc/bind/local.conf
zone "au-team.irpо" {
        type master;
        file "au.db";
};
zone "8.16.172. in-addr.arpa" {
        type master;
        file "0.db";
};
cd /etc/bind/zone
cp localdomain au.db
cp 127.in-addr.arpa 0.db
Chown root:named {au,0}.db
nano au.db
au-team.irpo. root.au-team.irpo.
moodle hq-rtr.au-team.irpo.
wiki hq-rtr.au-team.irpo.
nano 0.db
au-team.irpo. root.au-team.irpo.
1 IN PTR hq-rtr.au-team.irpo.
2 IN PTR hq-srv.au-team.irpo.
3 IN PTR hq-cli.au-team.irpo.
Systemctl restart bind
