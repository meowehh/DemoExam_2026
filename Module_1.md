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
- Реализовать на HQ-RTR маршрутизацию трафика всех указанных VLAN.
- использованием одного сетевого адаптера ВМ/физического порта.
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

> ⚠️ 💡 **Важно**: Хоть в задании (если смотреть на отчет и таблицу) не указано дать название ISP, но для корректного функционирования DNS и других сервисов требуется выдать полное доменное имя всем устройствам.

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
