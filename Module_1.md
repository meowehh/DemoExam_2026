# 🛠️ Руководство по выполнению демонстрационного экзамена CCA-2026

## Общие требования
- **Время выполнения:** 2 часа 30 минут (1 модуль = 1 час, 2 модуль = 1.5 часа)
- **Оборудование:** 10 серверных/Just OS машин (Alt JeOS, Alt Server), 2 клиентские машины (Alt Workstation), в моем случае все выполнялось на развернутом Proxmox внутри VMware Workstation.

--- 

# Модуль 1: Сетевое администрирование (1 час)

## 📋 Задание 1: Произведите базовую настройку устройств. + Задание 4: Настройте коммутацию в сегменте HQ.

**Задание 1**: 
- Настройте имена устройств согласно топологии. Используйте полное доменное имя.
- На всех устройствах необходимо сконфигурировать IPv4.
- IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918.
- Локальная сеть в сторону HQ-SRV(VLAN 100) должна вмещать не более 32 адресов.
- Локальная сеть в сторону HQ-CLI(VLAN 200) должна вмещать не менее 16 адресов.
- Локальная сеть для управления(VLAN 999) должна вмещать не более 8 адресов.
- Локальная сеть в сторону BR-SRV должна вмещать не более 16 адресов.
- Сведения об адресах занесите в таблицу 2, в качестве примера используйте Прил_3_О1_КОД 09.02.06-1-2026-М1.
  
**Задание 4**:
- Трафик HQ-SRV должен принадлежать VLAN 100.
- Трафик HQ-CLI должен принадлежать VLAN 200.
- Предусмотреть возможность передачи трафика управления в VLAN 999.
- Реализовать на HQ-RTR маршрутизацию трафика всех указанных VLAN использованием одного сетевого адаптера ВМ/физического порта.
- Сведения о настройке коммутации внесите в отчёт.

## Выполнение:
### Настройка hostname.
### ISP
```bash
hostnamectl set-hostname isp.au-team.irpo; exec bash
```
### HQ-RTR
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
```
### HQ-SRV
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
```
### HQ-CLI
```bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
```
### BR-RTR
```bash
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
```
### BR-SRV
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```
> [!IMPORTANT]
> ⚠️ 💡 **Важно**: Хоть в задании (если смотреть на отчет и таблицу) не указано дать название ISP, но его все равно нужно выдать.

> [!TIP]
>⚠️ **Примечание**: Команда hostnamectl set-hostname применяет изменения немедленно без перезагрузки. Флаг ; exec bash обновляет текущую сессию shell для отображения нового hostname в приглашении командной строки.

### Конфигурация IPv4 адресов.

### ISP
```bash
mkdir /etc/net/ifaces/enp7s2
mkdir /etc/net/ifaces/enp7s3
```
```bash
vim /etc/net/ifaces/enp7s2/options
BOOTPROTO=static
TYPE=eth
vim /etc/net/ifaces/enp7s2/ipv4address
172.16.1.1/28
```
```bash
vim /etc/net/ifaces/enp7s3/options
BOOTPROTO=static
TYPE=eth
vim /etc/net/ifaces/enp7s3/ipv4address
172.16.2.1/28
```
```bash
systemctl restart network
```
```bash
ip -c -br a
```
**Должен быть такой вывод у команды:**
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             192.168.120.157/24 fe80::be24:11ff:fe74:fa7/64 
enp7s2           UP             172.16.1.1/28 fe80::be24:11ff:fed1:a8dc/64 
enp7s3           UP             172.16.2.1/28 fe80::be24:11ff:fed6:e399/64
```
> [!NOTE] 
> ⚠️ 💡 Примечание: Для enp7s1 вывод будет отличаться из-за того что у всех этот интерфейс зависит от их собственной локальной сети, так как это интерфейс через который идет выход в интернет с помощью Bridge из Proxmox в VMware, в VMware обязательно нужно было указать Bridge в типе сетевого подключения, тип NAT или создание отдельной Network внутри VMware может вызывать нестабильность в работе!

### HQ-RTR
```bash
mkdir /etc/net/ifaces/enp7s2
mkdir /etc/net/ifaces/enp7s2.100
mkdir /etc/net/ifaces/enp7s2.200
mkdir /etc/net/ifaces/enp7s2.999
```
```bash
vim /etc/net/ifaces/enp7s1/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4address
172.16.1.2/28
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4route
default via 172.16.1.1
```
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 9.9.9.9
```
```bash
vim /etc/net/ifaces/enp7s2/options
BOOTPROTO=none
TYPE=eth
```
```bash
vim /etc/net/ifaces/enp7s2.100/options
BOOTPROTO=static
TYPE=vlan
VID=100
HOST=enp7s2
```
```bash
vim /etc/net/ifaces/enp7s2.100/ipv4address
192.168.100.1/27
```
```bash
vim /etc/net/ifaces/enp7s2.200/options
BOOTPROTO=static
TYPE=vlan
VID=200
HOST=enp7s2
```
```bash
vim /etc/net/ifaces/enp7s2.200/ipv4address
192.168.200.65/28
```
```bash
vim /etc/net/ifaces/enp7s2.999/options
BOOTPROTO=static
TYPE=vlan
VID=999
HOST=enp7s2
```
```bash
vim /etc/net/ifaces/enp7s2.999/ipv4address
192.168.99.89/29
```
```bash
systemctl restart network
```
```bash
ip -c -br a
```
**Должен быть такой вывод у команды:**
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             172.16.1.2/28 fe80::be24:11ff:feda:daba/64 
enp7s2           UP             fe80::be24:11ff:feae:ad50/64 
enp7s2.100@enp7s2 UP             192.168.100.1/27 fe80::be24:11ff:feae:ad50/64 
enp7s2.200@enp7s2 UP             192.168.200.65/28 fe80::be24:11ff:feae:ad50/64 
enp7s2.999@enp7s2 UP             192.168.99.89/29 fe80::be24:11ff:feae:ad50/64
```
> [!CAUTION]
> ⚠️ 💡 Важно!: Так как VLAN созданы через network внутри Proxmox, обязательно идем в веб панель Proxmox VE, заходим в раздел Server View > Datacenter > pve. В этом разделе в открытом списке выбираем 10103, 10104 машины (HQ-SRV,HQ-CLI), заходим в настройки во вкладку Hardware, меняем в графе Network Device (net6) VLAN tag, с того который там указан на 100 для HQ-CLI, и на 200 для HQ-SRV. Перезапускать машины не нужно.

### HQ-SRV
⚠️ 💡 Для enp7s1 (/etc/net/ifaces/enp7s1/options) в HQ-RTR, нужно заменить:
```bash
vim /etc/net/ifaces/enp7s1/options 
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```
На те параметры что указаны ниже:
```bash
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4address
192.168.100.2/27
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4route
default via 192.168.100.1
```
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 9.9.9.9
```
```bash
systemctl restart network
```
```bash
ip -c -br a
```
**Должен быть такой вывод у команды:**
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             192.168.100.2/27 fe80::be24:11ff:fef0:121/64 
```

### BR-RTR
```bash
mkdir /etc/net/ifaces/enp7s2/
```
```bash
vim /etc/net/ifaces/enp7s2/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/enp7s2/ipv4address
192.168.3.1/28
```
```bash
vim /etc/net/ifaces/enp7s1/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4address
172.16.2.2/28
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4route
default via 172.16.2.1
```
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 9.9.9.9
```
```bash
systemctl restart network
```
```bash
ip -c -br a
```
**Должен быть такой вывод у команды:**
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             172.16.2.2/28 fe80::be24:11ff:fe33:b6b2/64 
enp7s2           UP             192.168.3.1/28 fe80::be24:11ff:fea1:62b4/64
```

### BR-SRV
⚠️ 💡 Для enp7s1 (/etc/net/ifaces/enp7s1/options) в HQ-RTR, нужно заменить:
```bash
vim /etc/net/ifaces/enp7s1/options 
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```
На те параметры что указаны ниже:
```bash
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4address
192.168.3.2/28
```
```bash
vim /etc/net/ifaces/enp7s1/ipv4route
default via 192.168.3.1
```
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 9.9.9.9
```
```bash
systemctl restart network
```
```bash
ip -c -br a
```
**Должен быть такой вывод у команды:**
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             192.168.3.2/28 fe80::be24:11ff:fed0:f63a/64 
```
> [!TIP]
> ⚠️ 💡 Примечание!: HQ-CLI будет настроен позднее так как там будет использоваться DHCP настройка, на данном этапе теперь требуется настроить проброс портов чтобы пинг начал ходить между устройствами и появился доступ в интернет со всех машин, так же все отчеты будут приведны в отдельном [файле](./report_2026.odt), сейчас заполнять ничего не требуется, несмотря на задание.

## 📋 Задание 2: Настройте доступ к сети Интернет, на маршрутизаторе ISP + Задание 8: Настройка динамической трансляции адресов.

### Задание 2:
- Настройте адресацию на интерфейсах. **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2026/blob/main/Module_1.md#isp-1))**
- Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP **(Изначально так и есть, ничего делать не нужно)**
- Настройте маршрут по умолчанию, если это необходимо. **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2026/blob/main/Module_1.md#isp-1))**
- Настройте интерфейс, в сторону HQ-RTR, интерфейс подключен к сети 172.16.1.0/28 **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2026/blob/main/Module_1.md#isp-1))**
- Настройте интерфейс, в сторону BR-RTR, интерфейс подключен к сети 172.16.2.0/28 **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2026/blob/main/Module_1.md#isp-1))**
- На ISP настройте динамическую сетевую трансляцию портов для доступа к сети Интернет HQ-RTR и BR-RTR.
### Задание 8:
- Настройте динамическую трансляцию адресов для обоих офисов.
- Все устройства в офисах должны иметь доступ к сети Интернет

### ISP
```bash
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
systemctl restart network
```
```bash
apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -o enp7s1 -s 172.16.1.0/28 -j MASQUERADE
iptables -t nat -A POSTROUTING -o enp7s1 -s 172.16.2.0/28 -j MASQUERADE

iptables -A FORWARD -i ens19 -o enp7s1 -s 172.16.1.0/28 -j ACCEPT
iptables -A FORWARD -i ens20 -o enp7s1 -s 172.16.2.0/28 -j ACCEPT

iptables-save > /etc/sysconfig/iptables
systemctl enable iptables --now
systemctl restart iptables
```
```bash
systemctl status iptables
iptables -t nat -L -n -v
```
Должны быть такие выводы у команд:
```bash
● iptables.service - IPv4 firewall with iptables
     Loaded: loaded (/lib/systemd/system/iptables.service; enabled; vendor preset: disabled)
     Active: active (exited) since Tue 2025-12-09 11:07:03 +07; 5s ago
    Process: 8199 ExecStart=/etc/init.d/iptables start (code=exited, status=0/SUCCESS)
   Main PID: 8199 (code=exited, status=0/SUCCESS)
        CPU: 11ms

Dec 09 11:07:03 isp.au-team.irpo systemd[1]: Starting IPv4 firewall with iptables...
Dec 09 11:07:03 isp.au-team.irpo iptables[8213]: Applying iptables firewall rules: succeeded
Dec 09 11:07:03 isp.au-team.irpo iptables[8199]: Applying iptables firewall rules: [ DONE ]
Dec 09 11:07:03 isp.au-team.irpo systemd[1]: Finished IPv4 firewall with iptables.

Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      enp7s1  172.16.1.0/28        0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      enp7s1  172.16.2.0/28        0.0.0.0/0 
```
> [!TIP]
> ⚠️ 💡 **Примечание!**: Сразу же настроим интернет на всех устройствах, для этого потребуется повтороить настройку на всех устройствах, детали приведены ниже.

### HQ-RTR
```bash
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
systemctl restart network
```
```bash
apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -o enp7s1 -s 192.168.100.0/27 -j MASQUERADE
iptables -t nat -A POSTROUTING -o enp7s1 -s 192.168.200.64/28 -j MASQUERADE
iptables -t nat -A POSTROUTING -o enp7s1 -s 192.168.99.88/29 -j MASQUERADE

iptables -A FORWARD -i ens19.10 -o enp7s1 -s 192.168.100.0/27 -j ACCEPT
iptables -A FORWARD -i ens19.20 -o enp7s1 -s 192.168.200.64/28 -j ACCEPT
iptables -A FORWARD -i ens19.99 -o enp7s1 -s 192.168.99.88/29 -j ACCEPT

iptables-save > /etc/sysconfig/iptables
systemctl enable iptables --now
systemctl restart iptables
```
```bash
systemctl status iptables
iptables -t nat -L -n -v
```
Должны быть такие выводы у команд:
```bash
● iptables.service - IPv4 firewall with iptables
     Loaded: loaded (/lib/systemd/system/iptables.service; enabled; vendor preset: disabled)
     Active: active (exited) since Tue 2025-12-09 04:10:30 UTC; 4s ago
    Process: 8484 ExecStart=/etc/init.d/iptables start (code=exited, status=0/SUCCESS)
   Main PID: 8484 (code=exited, status=0/SUCCESS)
        CPU: 11ms

Dec 09 04:10:30 hq-rtr.au-team.irpo systemd[1]: Starting IPv4 firewall with iptables...
Dec 09 04:10:30 hq-rtr.au-team.irpo iptables[8498]: Applying iptables firewall rules: succeeded
Dec 09 04:10:30 hq-rtr.au-team.irpo iptables[8484]: Applying iptables firewall rules: [ DONE ]
Dec 09 04:10:30 hq-rtr.au-team.irpo systemd[1]: Finished IPv4 firewall with iptables.

Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      enp7s1  192.168.100.0/27     0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      enp7s1  192.168.200.64/28    0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      enp7s1  192.168.99.88/29     0.0.0.0/0
```
### BR-RTR
```bash
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
systemctl restart network
```
```bash
apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -o enp7s1 -s 192.168.3.0/28 -j MASQUERADE
iptables -A FORWARD -i ens19 -o enp7s1 -s 192.168.3.0/28 -j ACCEPT

iptables-save > /etc/sysconfig/iptables

systemctl enable iptables --now
systemctl restart iptables
```
```bash
systemctl status iptables
iptables -t nat -L -n -v
```
Должны быть такие выводы у команд:
```bash
● iptables.service - IPv4 firewall with iptables
     Loaded: loaded (/lib/systemd/system/iptables.service; enabled; vendor preset: disabled)
     Active: active (exited) since Tue 2025-12-09 04:12:17 UTC; 5s ago
    Process: 7872 ExecStart=/etc/init.d/iptables start (code=exited, status=0/SUCCESS)
   Main PID: 7872 (code=exited, status=0/SUCCESS)
        CPU: 11ms

Dec 09 04:12:17 br-rtr.au-team.irpo systemd[1]: Starting IPv4 firewall with iptables...
Dec 09 04:12:17 br-rtr.au-team.irpo iptables[7887]: Applying iptables firewall rules: succeeded
Dec 09 04:12:17 br-rtr.au-team.irpo iptables[7872]: Applying iptables firewall rules: [ DONE ]
Dec 09 04:12:17 br-rtr.au-team.irpo systemd[1]: Finished IPv4 firewall with iptables.

Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      enp7s1  192.168.3.0/28       0.0.0.0/0 
```
> [!IMPORTANT] 
>⚠️ **Важно**: На данном этапе уже должен работать выход в Интернет на всех устройствах (кроме HQ-CLI, его настроим позже по DHCP), а также пинг между ними. Если что-то не работает, значит где-то ошибка.

## 📋 Задание 11: Настройте часовой пояс на всех устройствах (за исключением виртуального коммутатора, в случае его использования) согласно месту проведения экзамена.

### ISP, HQ-RTR, BR-RTR
**На данных устройствах необходимо предварительно установить пакет tzdata для некоторых регионов которых нет по умолчанию на Alt JeOS, в 2026 году это ISP, HQ-RTR, BR-RTR. Если для экзамена использовались машины из скрипта PavelAF.**
```bash
apt-get update && apt-get install tzdata -y
timedatectl set-timezone Asia/Novosibirsk
timedatectl # Проверка 
```
### HQ-SRV, HQ-CLI, BR-SRV
**На данных устройствах можно сразу установить часовой пояс, так как тут используется Alt Server и Alt Workstation, где уже есть пакет tzdata.**
```bash
timedatectl set-timezone Asia/Novosibirsk
timedatectl # Проверка 
```


## 📋 Задание 3: Создание локальных учетных записей.

### Задание 3:
- Создайте локальные учетные записи на серверах HQ-SRV и BR-SRV:
- Создайте пользователя sshuser.
- Пароль пользователя sshuser с паролем P@ssw0rd.
- Идентификатор пользователя 2026.
- Пользователь sshuser должен иметь возможность запускать sudo без ввода пароля.
- Создайте пользователя net_admin на маршрутизаторах HQ-RTR и BR-RTR.
- Пароль пользователя net_admin с паролем P@ssw0rd.
- При настройке ОС на базе Linux, запускать sudo без ввода пароля.
- При настройке ОС отличных от Linux пользователь должен обладать максимальными привилегиями.

### HQ-SRV и BR-SRV
```bash
useradd sshuser -u 2026 -U
passwd sshuser # P@ssw0rd
usermod -a -G wheel sshuser

vim /etc/sudoers
## Same thing without a password
# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
sshuser ALL=(ALL) NOPASSWD: ALL
```
Из под нового пользователя sshuser должен быть доступ без пароля:
```bash
exit
net_admin # P@ssw0rd
sudo cat /root/.bashrc
```

### HQ-RTR и BR-RTR
```bash
apt-get update && apt-get install sudo -y
```
```bash
useradd net_admin
passwd net_admin # P@ssw0rd
usermod -a -G wheel net_admin

vim /etc/sudoers
## Same thing without a password
# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
net_admin ALL=(ALL) NOPASSWD: ALL
```
Из под нового пользователя net_admin должен быть доступ без пароля:
```bash
exit
net_admin # P@ssw0rd
sudo cat /root/.bashrc
```

## 📋 Задание 5: Настройте безопасный удаленный доступ на серверах HQ-SRV и BR-SRV.

- Для подключения используйте порт 2026.
- Разрешите подключения только пользователю sshuser.
- Ограничьте количество попыток входа до двух.
- Настройте баннер «Authorized access only».

### BR-SRV
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 2026
MaxAuthTries 2
Banner /etc/openssh/sshd_banner
AllowUsers sshuser
```
```bash
vim /etc/openssh/sshd_banner
«Authorized access only»
```
```bash
systemctl enable sshd --now
systemctl restart sshd

systemctl status sshd
```
Должен быть такой вывод у команды:
```bash
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2025-12-09 23:32:23 +07; 4s ago
    Process: 3075 ExecStartPre=/usr/bin/ssh-keygen -A (code=exited, status=0/SUCCESS)
    Process: 3077 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 3078 (sshd)
      Tasks: 1 (limit: 1131)
     Memory: 748.0K
        CPU: 6ms
     CGroup: /system.slice/sshd.service
             └─ 3078 /usr/sbin/sshd -D

Dec 09 23:32:23 br-srv.au-team.irpo systemd[1]: Starting OpenSSH server daemon...
Dec 09 23:32:23 br-srv.au-team.irpo sshd[3078]: Server listening on 0.0.0.0 port 2026.
Dec 09 23:32:23 br-srv.au-team.irpo systemd[1]: Started OpenSSH server daemon.
Dec 09 23:32:23 br-srv.au-team.irpo sshd[3078]: Server listening on :: port 2026.
```
```bash
ssh sshuser@localhost -p 2026
```
Должен быть такой вывод у команды:
```bash
The authenticity of host '[localhost]:2026 ([127.0.0.1]:2026)' can't be established.
ED25519 key fingerprint is SHA256:I5hoQPp6etwA1OX7wCOKfAFLhiJ8U848g4KUIWRnjyY.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:2026' (ED25519) to the list of known hosts.
«Authorized access only»

sshuser@localhost's password: 
Last login: Tue Dec  9 23:04:42 2025
[sshuser@br-srv ~]$ 
```

### HQ-SRV
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 2026
MaxAuthTries 2
Banner /etc/openssh/sshd_banner
AllowUsers sshuser
```
```bash
vim /etc/openssh/sshd_banner
«Authorized access only»
```
```bash
systemctl enable sshd --now
systemctl restart sshd

systemctl status sshd
```
Должен быть такой вывод у команды:
```bash
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2025-12-09 23:25:42 +07; 1s ago
    Process: 3252 ExecStartPre=/usr/bin/ssh-keygen -A (code=exited, status=0/SUCCESS)
    Process: 3254 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 3255 (sshd)
      Tasks: 1 (limit: 1131)
     Memory: 744.0K
        CPU: 6ms
     CGroup: /system.slice/sshd.service
             └─ 3255 /usr/sbin/sshd -D

Dec 09 23:25:42 hq-srv.au-team.irpo systemd[1]: Starting OpenSSH server daemon...
Dec 09 23:25:42 hq-srv.au-team.irpo systemd[1]: Started OpenSSH server daemon.
Dec 09 23:25:42 hq-srv.au-team.irpo sshd[3255]: Server listening on 0.0.0.0 port 2026.
Dec 09 23:25:42 hq-srv.au-team.irpo sshd[3255]: Server listening on :: port 2026.
```
```bash
ssh sshuser@localhost -p 2026
```
Должен быть такой вывод у команды:
```bash
The authenticity of host '[localhost]:2026 ([127.0.0.1]:2026)' can't be established.
ED25519 key fingerprint is SHA256:ozykrFl1QDnyY+S2wnNx+ZVlUyxY3ct74Bj4RVkmNnI.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:2026' (ED25519) to the list of known hosts.
«Authorized access only»

sshuser@localhost's password: 
Last login: Tue Dec  9 23:03:21 2025
[sshuser@hq-srv ~]$ 
```
> [!WARNING]
>⚠️ 💡 Примечание: В файле баннера /etc/openssh/sshd_banner, нужно поставить 1-2 отступа вниз чтобы баннер корректно отображался, в завимости от редактора vim/nano. Иначе баннер будет наезжать на поле авторизации или на строку приглашения.

## 📋 Задание 6: Между офисами HQ и BR, на маршрутизаторах HQ-RTR и BR-RTR необходимо сконфигурировать ip туннель.

- На выбор технологии GRE или IP in IP.
- Сведения о туннеле занесите в отчёт. (Отчет будет приложен отдельным [файлом](./report_2026.odt))

### HQ-RTR
```bash
mkdir /etc/net/ifaces/gre1
```
```bash
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.1.2
TUNREMOTE=172.16.2.2
TUNOPTIONS='ttl 64'
```
```bash
vim /etc/net/ifaces/gre1/ipv4address
10.10.0.1/30
```
```bash
systemctl restart network
ip -c -br a
```
Если все сделано верно получаем следующий вывод:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             172.16.1.2/28 fe80::be24:11ff:feda:daba/64 
enp7s2           UP             fe80::be24:11ff:feae:ad50/64 
enp7s2.100@enp7s2 UP             192.168.100.1/27 fe80::be24:11ff:feae:ad50/64 
enp7s2.200@enp7s2 UP             192.168.200.65/28 fe80::be24:11ff:feae:ad50/64 
enp7s2.999@enp7s2 UP             192.168.99.89/29 fe80::be24:11ff:feae:ad50/64 
gre0@NONE        DOWN           
gretap0@NONE     DOWN           
erspan0@NONE     DOWN           
gre1@NONE        UNKNOWN        10.10.0.1/30 fe80::ac10:102/64 
```

### BR-RTR
```bash
mkdir /etc/net/ifaces/gre1
```
```bash
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.2.2
TUNREMOTE=172.16.1.2
TUNOPTIONS='ttl 64'
```
```bash
vim /etc/net/ifaces/gre1/ipv4address
10.10.0.2/30
```
```bash
systemctl restart network
ip -c -br a
```
Если все сделано верно получаем следующий вывод:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             172.16.2.2/28 fe80::be24:11ff:fe33:b6b2/64 
enp7s2           UP             192.168.3.1/28 fe80::be24:11ff:fea1:62b4/64 
gre0@NONE        DOWN           
gretap0@NONE     DOWN           
erspan0@NONE     DOWN           
gre1@NONE        UNKNOWN        10.10.0.2/30 fe80::ac10:202/64 
```
> [!NOTE]
> **Проверка**: Проверяем работоспобность пингуя по туннелю с 10.10.0.1 на 10.10.0.2 и обратно.

## 📋 Задание 7: Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической маршрутизации используйте link state протокол на ваше усмотрение.

- Разрешите выбранный протокол только на интерфейсах в ip туннеле.
- Маршрутизаторы должны делиться маршрутами только друг с другом.
- Обеспечьте защиту выбранного протокола посредством парольной защиты.
- Сведения о настройке и защите протокола занесите в отчёт. ([Отдельный файл](./report_2026.odt))

### HQ-RTR
```bash
apt-get update && apt-get install frr -y
```
```bash
vim /etc/frr/daemons
ospfd=yes
```
```bash
systemctl enable --now frr
systemctl restart frr
reboot
```
```bash
vtysh
show run
```
**Вывод:**
```bash
Current configuration:
!
frr version 9.0.2
frr defaults traditional
hostname hq-rtr.au-team.irpo
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface gre1
 ip ospf network broadcast
exit
!
end
```
> [!CAUTION]
> Если все было настроено верно (интерфейс gre,ospfd и нигде не было ошибки) получаем такой вывод, самое главное у вас сама по себе должна появиться строка с интрефейсом, не нужно создавать его самим на FRR, нужно выполнить все в точности как у меня, если интерфейс gre1 внутри FRR создается сам, отсекается большая часть проблем.

### BR-RTR
```bash
apt-get update && apt-get install frr -y
```
```bash
vim /etc/frr/daemons
ospfd=yes
```
```bash
systemctl enable --now frr
systemctl restart frr
reboot
```
```bash
vtysh
show run
```
```bash
Current configuration:
!
frr version 9.0.2
frr defaults traditional
hostname br-rtr.au-team.irpo
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface gre1
 ip ospf network broadcast
exit
!
end
```
### HQ-RTR
```bash
hq-rtr.au-team.irpo# conf t
hq-rtr.au-team.irpo(config)# router ospf
hq-rtr.au-team.irpo(config-router)# ospf router-id 172.16.1.1
hq-rtr.au-team.irpo(config-router)# network 10.10.0.0/30 area 0
hq-rtr.au-team.irpo(config-router)# network 192.168.100.0/27 area 0
hq-rtr.au-team.irpo(config-router)# network 192.168.200.64/28 area 0
hq-rtr.au-team.irpo(config-router)# network 192.168.99.88/29 area 0
hq-rtr.au-team.irpo(config-router)# area 0 authentication
hq-rtr.au-team.irpo(config-router)# exit
hq-rtr.au-team.irpo(config)# interface gre1 
hq-rtr.au-team.irpo(config-if)# ip ospf authentication-key P@ssw0rd
hq-rtr.au-team.irpo(config-if)# ip ospf authentication             
hq-rtr.au-team.irpo(config-if)# no ip ospf passive
hq-rtr.au-team.irpo(config-if)# exit
hq-rtr.au-team.irpo(config)# exit
hq-rtr.au-team.irpo# wr
```
```bash
show run
```
**Содержание конфигурации FRR после настройки:**
```bash
Current configuration:
!
frr version 9.0.2
frr defaults traditional
hostname hq-rtr.au-team.irpo
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface gre1
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 ip ospf network broadcast
 no ip ospf passive
exit
!
router ospf
 ospf router-id 172.16.1.1
 network 10.10.0.0/30 area 0
 network 192.168.99.88/29 area 0
 network 192.168.100.0/27 area 0
 network 192.168.200.64/28 area 0
 area 0 authentication
exit
!
end
```
### BR-RTR
```bash
br-rtr.au-team.irpo# conf t
br-rtr.au-team.irpo(config)# router ospf
br-rtr.au-team.irpo(config-router)# ospf router-id 172.16.2.1
br-rtr.au-team.irpo(config-router)# network 10.10.0.0/30 area 0
br-rtr.au-team.irpo(config-router)# network 192.168.3.0/28 area 0
br-rtr.au-team.irpo(config-router)# area 0 authentication 
br-rtr.au-team.irpo(config-router)# exit
br-rtr.au-team.irpo(config)# interface gre1
br-rtr.au-team.irpo(config-if)# ip ospf authentication-key P@ssw0rd
br-rtr.au-team.irpo(config-if)# ip ospf authentication             
br-rtr.au-team.irpo(config-if)# no ip ospf passive
br-rtr.au-team.irpo(config-if)# exit
br-rtr.au-team.irpo(config)# exit
br-rtr.au-team.irpo# wr
```
```bash
show run
```
**Содержание конфигурации FRR после настройки:**
```bash
Current configuration:
!
frr version 9.0.2
frr defaults traditional
hostname br-rtr.au-team.irpo
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface gre1
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 ip ospf network broadcast
 no ip ospf passive
exit
!
router ospf
 ospf router-id 172.16.2.1
 network 10.10.0.0/30 area 0
 network 192.168.3.0/28 area 0
 area 0 authentication
exit
!
end
```
Проверим работоспобность OSPF, для этого воспользуемся информацией о соседях полученных через OSPF, состояние должно быть Full/DR,Full/Backup.
```bash
hq-rtr.au-team.irpo# show ip ospf neighbor 
Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
172.16.2.1        1 Full/Backup     1m01s             38.172s 10.10.0.2       gre1:10.10.0.1                       0     0     0

```
```bash
br-rtr.au-team.irpo# show ip ospf neighbor
Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
172.16.1.1        1 Full/DR         1m05s             34.321s 10.10.0.1       gre1:10.10.0.2                       0     0     0
```
> [!NOTE] 
>⚠️ 💡 Примечание: После того как OSPF успешно работает, нужно проверить пинг, напимер с HQ-SRV попробовать пинговать BR-SRV и обратно, пинг должен успешно проходить между любыми устройствами, кроме ISP и пока не настроенного HQ-CLI.

## 📋 Задание 9: Настройте протокол динамической конфигурации хостов для сети в сторону HQ-CLI.

**Задание 9**:
- Настройть нужную подсеть.
- Для офиса HQ в качестве сервера DHCP выступает маршрутизатор HQ-RTR.
- Клиентом является машина HQ-CLI.
- Исключить из выдачи адрес маршрутизатора.
- Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR.
- Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV.
- DNS-суффикс для офисов HQ – au-team.irpo
- Сведения о настройке протокола занесите в [отчёт](./report_2025.docx)
  
### HQ-RTR
```bash
apt-get update && apt-get install dhcp-server nano -y #Рекомендуется настраивать через nano для корретной табуляции внутри dhcpd.conf.
nano /etc/dhcp/dhcpd.conf.sample #Взять шаблон конфига настроек можно отсюда или готовый ниже.
```
**Готовый конфиг**:
```bash
nano /etc/dhcp/dhcpd.conf
subnet 192.168.200.64 netmask 255.255.255.240 {
        option routers                  192.168.200.65;
        option subnet-mask              255.255.255.240;

        option domain-name              "au-team.irpo";
        option domain-name-servers      192.168.100.2;

        range dynamic-bootp 192.168.200.66 192.168.200.78;
        default-lease-time 600;
        max-lease-time 7200;
}
```
```bash
systemctl enable --now dhcpd
systemctl restart dhcpd
```
> [!NOTE] 
> С настройкой DHCP-сервера закончено, теперь получим IP если HQ-CLI ещё этого не сделал сам.

### HQ-CLI
```bash
dhcpcd
```
Вывод у команды должен быть таким:
```bash
dhcpcd-9.4.0 starting
DUID 00:04:a4:4f:22:43:ad:81:49:e1:b2:c7:06:fb:19:ec:1c:6a
enp7s1: soliciting a DHCP lease
enp7s1: offered 192.168.200.66 from 192.168.200.65
enp7s1: leased 192.168.200.66 for 600 seconds
enp7s1: adding route to 192.168.200.64/28
enp7s1: adding default route via 192.168.200.65
forked to background, child pid 2593
```
```bash
ip -c -br a
```
**Вывод:**
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp7s1           UP             192.168.200.66/28 fe80::be24:11ff:fec6:63e9/64 
```

### HQ-RTR

**Проверка службы на возможные ошибки**:
```bash
systemctl status dhcpd
```
Вывод должен быть таким:
```bash
● dhcpd.service - DHCPv4 Server Daemon
     Loaded: loaded (/lib/systemd/system/dhcpd.service; enabled; vendor preset: disabled)
     Active: active (running) since Wed 2025-12-10 10:12:20 +07; 2min 8s ago
       Docs: man:dhcpd(8)
             man:dhcpd.conf(5)
    Process: 3338 ExecStartPre=/etc/chroot.d/dhcpd.all (code=exited, status=0/SUCCESS)
   Main PID: 3419 (dhcpd)
      Tasks: 1 (limit: 529)
     Memory: 4.3M
        CPU: 40ms
     CGroup: /system.slice/dhcpd.service
             └─ 3419 /usr/sbin/dhcpd -4 -f --no-pid

Dec 10 10:12:20 hq-rtr.au-team.irpo dhcpd[3419]: Wrote 0 leases to leases file.
Dec 10 10:12:20 hq-rtr.au-team.irpo dhcpd[3419]: Server starting service.
Dec 10 10:12:22 hq-rtr.au-team.irpo dhcpd[3419]: DHCPDISCOVER from bc:24:11:c6:63:e9 via enp7s2.200
Dec 10 10:12:23 hq-rtr.au-team.irpo dhcpd[3419]: DHCPOFFER on 192.168.200.66 to bc:24:11:c6:63:e9 (hq-cli) via enp7s2.200
Dec 10 10:12:23 hq-rtr.au-team.irpo dhcpd[3419]: DHCPREQUEST for 192.168.200.66 (192.168.200.65) from bc:24:11:c6:63:e9 (hq-cli) via enp7s2.200
Dec 10 10:12:23 hq-rtr.au-team.irpo dhcpd[3419]: DHCPACK on 192.168.200.66 to bc:24:11:c6:63:e9 (hq-cli) via enp7s2.200
```
> [!NOTE] 
>⚠️ 💡 **Примечание**: После этого пинг до интернета должен заработать, можно проверрить до 1.1.1.1, пинг по доменным именам пока что не работает, так как локальный DNS на HQ-SRV будет настроен ниже.

> [!CAUTION]
>⚠️ **Важно**: В случае перезапуска сетевой службы network на HQ-RTR, необходимо в ручную перезапускать каждый раз службу dhcpd (systemctl restart dhcpd), так как она будет сыпать ошибками и выключаться.

## 📋 Задание 10: Настройте инфраструктуру разрешения доменных имён для офисов HQ и BR.

**Задание 10**:
- Основной DNS-сервер реализован на HQ-SRV
- Сервер должен обеспечивать разрешение имён в сетевые адреса устройств и обратно в соответствии с таблицей 3
- В качестве DNS сервера пересылки используйте любой общедоступный DNS сервер.

**Таблица 3:**
```bash
Устройство Запись Тип
HQ-RTR hq-rtr.au-team.irpo A,PTR
BR-RTR br-rtr.au-team.irpo A
HQ-SRV hq-srv.au-team.irpo A,PTR
HQ-CLI hq-cli.au-team.irpo A,PTR
BR-SRV br-srv.au-team.irpo A
ISP (интерфейс направленный в сторону HQ-RTR) docker.au-team.irpo A
ISP (интерфейс направленный в сторону BR-RTR) web.au-team.irpo A
```

### HQ-SRV
```bash
apt-get update && apt-get install bind nano -y
```
```bash
nano /etc/bind/options.conf
listen-on { any; };
forward first;
forwarders {9.9.9.9; };
allow-query { any; };
```
```bash
nano /etc/bind/local.conf
// Add other zones here
// Зона прямого просмотра (A-записи)
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

// Зона обратного просмотра для сети 192.168.100.0
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};

// Зона обратного просмотра для сети 192.168.200.64
zone "64.200.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.200.64";
};

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```
```bash
nano /etc/bind/db.au-team.irpo
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2025121001 ; serial
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400 )    ; minimum

    IN  NS  hq-srv.au-team.irpo.

hq-rtr   IN  A   192.168.100.1
br-rtr   IN  A   192.168.3.1
hq-srv   IN  A   192.168.100.2
hq-cli   IN  A   192.168.200.66
br-srv   IN  A   192.168.3.2

docker  IN   A   172.16.1.1
web     IN   A   172.16.2.1

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```
```bash
nano /etc/bind/db.192.168.100
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2025121001
        3600
        1800
        604800
        86400 )

    IN  NS  hq-srv.au-team.irpo.

1   IN  PTR  hq-rtr.au-team.irpo.
2   IN  PTR  hq-srv.au-team.irpo.

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```
```bash
nano /etc/bind/db.192.168.200.64
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2025121001
        3600
        1800
        604800
        86400 )

    IN  NS  hq-srv.au-team.irpo.

1  IN  PTR  hq-cli.au-team.irpo.

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```

```bash
rm -rf /etc/net/ifaces/enp7s1/resolv.conf 
systemctl restart network
```
```bash
nano /etc/resolvconf.conf
name_servers=127.0.0.1
```
```bash
resolvconf -u
systemctl restart network
```
**Выполним проверку**:
```bash
cat /etc/resolv.conf | grep nameserver
```
**Если все настроено верно получаем такой ответ**:
```bash
nameserver 127.0.0.1
```
**Запускаем службу DNS**: 
```bash
systemctl enable --now bind
systemctl restart bind
```
```bash
systemctl status bind
```
```bash
● bind.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/lib/systemd/system/bind.service; enabled; vendor preset: disabled)
     Active: active (running) since Wed 2025-12-10 10:34:08 +07; 4s ago
    Process: 16785 ExecStartPre=/etc/init.d/bind rndc_keygen (code=exited, status=0/SUCCESS)
    Process: 16790 ExecStartPre=/usr/sbin/named-checkconf $CHROOT -z /etc/named.conf (code=exited, status=0/SUCCESS)
    Process: 16791 ExecStart=/usr/sbin/named -u named $CHROOT $RETAIN_CAPS $EXTRAOPTIONS (code=exited, status=0/SUCCESS)
      Tasks: 5 (limit: 1131)
     Memory: 11.0M
        CPU: 23ms
     CGroup: /system.slice/bind.service
             └─ 16792 /usr/sbin/named -u named

Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:500:9f::42#53
Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:dc3::35#53
Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:503:c27::2:30#53
Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:500:1::53#53
Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:500:2f::f#53
Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:500:2::c#53
Dec 10 10:34:08 hq-srv.au-team.irpo named[16792]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
Dec 10 10:34:09 hq-srv.au-team.irpo named[16792]: managed-keys-zone: Key 20326 for zone . is now trusted (acceptance timer complete)
Dec 10 10:34:09 hq-srv.au-team.irpo named[16792]: managed-keys-zone: Key 38696 for zone . is now trusted (acceptance timer complete)
Dec 10 10:34:09 hq-srv.au-team.irpo named[16792]: resolver priming query complete
```
> [!IMPORTANT] 
>⚠️ **Важно**: Проверяем с помощью пинга соседей по их доменным именам, пробуем пинговать br-srv.au-team.irpo, hq-cli.au-team.irpo, moodle.au-team.irpo, wiki.au-team.irpo и так далее. Проверяем выход в Интернет. Далее небходимо настроить этот локальный DNS сервер для всех машин, так как на hq-cli настроен DHCP где уже прописан этот сервер, то это будет нужно сделать только на HQ-RTR,BR-RTR,BR-SRV.

### HQ-RTR
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 192.168.100.2 # Старую запись удаляем, оставляем только новую.
```
```bash
systemctl restart network
systemctl restart dhcpd
```
### BR-RTR
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 192.168.100.2 # Старую запись удаляем, оставляем только новую.
```
```bash
systemctl restart network
```
### BR-SRV
```bash
vim /etc/net/ifaces/enp7s1/resolv.conf
nameserver 192.168.100.2 # Старую запись удаляем, оставляем только новую.
```
```bash
systemctl restart network
```
### HQ-RTR,HQ-CLI,HQ-SRV,BR-RTR,BR-SRV
```bash
ping hq-rtr.au-team.irpo
ping hq-cli.au-team.irpo
ping hq-srv.au-team.irpo
ping br-rtr.au-team.irpo
ping br-srv.au-team.irpo
ping web.au-team.irpo
ping docker.au-team.irpo
```
**Вывод (на каждой машине будет немного отличаться для тех узлов для которых по заданию нет PTR записи, главное чтобы был ответ):**
```bash
PING hq-rtr.au-team.irpo (192.168.100.1) 56(84) bytes of data.
64 bytes from hq-rtr.au-team.irpo (192.168.100.1): icmp_seq=1 ttl=63 time=0.988 ms
64 bytes from hq-rtr.au-team.irpo (192.168.100.1): icmp_seq=2 ttl=63 time=1.21 ms
64 bytes from hq-rtr.au-team.irpo (192.168.100.1): icmp_seq=3 ttl=63 time=1.21 ms
64 bytes from hq-rtr.au-team.irpo (192.168.100.1): icmp_seq=4 ttl=63 time=0.940 ms

PING hq-srv.au-team.irpo (192.168.100.2) 56(84) bytes of data.
64 bytes from hq-srv.au-team.irpo (192.168.100.2): icmp_seq=1 ttl=62 time=0.950 ms
64 bytes from hq-srv.au-team.irpo (192.168.100.2): icmp_seq=2 ttl=62 time=1.32 ms
64 bytes from hq-srv.au-team.irpo (192.168.100.2): icmp_seq=3 ttl=62 time=1.24 ms
64 bytes from hq-srv.au-team.irpo (192.168.100.2): icmp_seq=4 ttl=62 time=1.31 ms

PING hq-cli.au-team.irpo (192.168.200.66) 56(84) bytes of data.
64 bytes from 192.168.200.66 (192.168.200.66): icmp_seq=1 ttl=62 time=1.08 ms
64 bytes from 192.168.200.66 (192.168.200.66): icmp_seq=2 ttl=62 time=1.29 ms
64 bytes from 192.168.200.66 (192.168.200.66): icmp_seq=3 ttl=62 time=1.24 ms
64 bytes from 192.168.200.66 (192.168.200.66): icmp_seq=4 ttl=62 time=1.23 ms

PING br-rtr.au-team.irpo (192.168.3.1) 56(84) bytes of data.
64 bytes from 192.168.3.1 (192.168.3.1): icmp_seq=1 ttl=63 time=0.993 ms
64 bytes from 192.168.3.1 (192.168.3.1): icmp_seq=2 ttl=63 time=1.22 ms
64 bytes from 192.168.3.1 (192.168.3.1): icmp_seq=3 ttl=63 time=1.29 ms
64 bytes from 192.168.3.1 (192.168.3.1): icmp_seq=4 ttl=63 time=1.11 ms

PING br-srv.au-team.irpo (192.168.3.2) 56(84) bytes of data.
64 bytes from 192.168.3.2 (192.168.3.2): icmp_seq=1 ttl=62 time=1.18 ms
64 bytes from 192.168.3.2 (192.168.3.2): icmp_seq=2 ttl=62 time=1.29 ms
64 bytes from 192.168.3.2 (192.168.3.2): icmp_seq=3 ttl=62 time=1.34 ms
64 bytes from 192.168.3.2 (192.168.3.2): icmp_seq=4 ttl=62 time=1.35 ms

PING docker.au-team.irpo (172.16.1.1) 56(84) bytes of data.
64 bytes from 172.16.1.1 (172.16.1.1): icmp_seq=1 ttl=63 time=0.544 ms
64 bytes from 172.16.1.1 (172.16.1.1): icmp_seq=2 ttl=63 time=0.737 ms
64 bytes from 172.16.1.1 (172.16.1.1): icmp_seq=3 ttl=63 time=0.983 ms
64 bytes from 172.16.1.1 (172.16.1.1): icmp_seq=4 ttl=63 time=0.790 ms

PING web.au-team.irpo (172.16.2.1) 56(84) bytes of data.
64 bytes from 172.16.2.1 (172.16.2.1): icmp_seq=1 ttl=63 time=0.551 ms
64 bytes from 172.16.2.1 (172.16.2.1): icmp_seq=2 ttl=63 time=0.905 ms
64 bytes from 172.16.2.1 (172.16.2.1): icmp_seq=3 ttl=63 time=0.905 ms
64 bytes from 172.16.2.1 (172.16.2.1): icmp_seq=4 ttl=63 time=0.810 ms
```
> [!NOTE]
> Проверямем пинг до Интернета, локальных доменных имен, все должно работать, со всех машин на все машины.

> [!TIP]
>После этих манипуляций, - Модуль 1: полностью выполнен, необходимо заполнить отчет как указано [здесь.](./report_2026.odt)
