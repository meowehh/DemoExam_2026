# Модуль 2: Системное администрирование (1.5 часа)

## 📋 Задание 1: Настройте контроллер домена Samba DC на сервере BR-SRV.

**Задание 1:**
- Имя домена au-team.irpo
- Введите в созданный домен машину HQ-CLI
- Создайте 5 пользователей для офиса HQ: имена пользователей формата hquser№ (например hquser1, hquser2 и т.д.)
- Создайте группу hq, введите в группу созданных пользователей
- Убедитесь, что пользователи группы hq имеют право аутентифицироваться на HQ-CLI
- Пользователи группы hq должны иметь возможность повышать привилегии для выполнения ограниченного набора команд: cat, grep, id.
- Запускать другие команды с повышенными привилегиями пользователи группы права не имеют.

### BR-SRV
Изначально на BR-SRV не прописан DNS сервер, нужно указать его для возможности загрузки пакетов и работы Samba DC.
```bash
vim /etc/resolvconf.conf
name_servers=127.0.0.1
name_servers=192.168.1.10
resolvconf -u
systemctl restart network
```
```bash
apt-get update && apt-get install -y task-samba-dc alterator-fbi alterator-net-domain admx-* admc gpui
```
```bash
vim /etc/sysconfig/network
HOSTNAME=br-srv.au-team.irpo
```
```bash
reboot
```
```bash
systemctl enable --now ahttpd alteratord
rm -rf /etc/samba/smb.conf /var/{lib.cache}/samba
mkdir -p /var/lib/samba/sysvol
```
```bash
samba-tool domain provision --realm=au-team.irpo --domain=au-team --adminpass='P@ssw0rd' --dns-backend=BIND9_DLZ --server-role=dc --use-rfc2307 
```

### HQ-CLI
```bash
apt-get update && apt-get install -y admx-* admc gpui sudo gpupdate
```
**Заходим через вкладку Console внутри Proxmox VE, чтобы получить доступ к графической части.**
```
login: user
password: resu
```
**Открываем Firefox:** 
- Переходим по адресу 192.168.3.10:8080
- Данные для авторизации:
```bash
login: root
password: toor
```
- Configuration > Expert mode > Apply
- Web Interface
- Меняем порт 8080 на 8081 > Apply > Restart http server
- Переходим по адресу 192.168.3.10:8081
- Вкладка Domain
- Выбираем Active Directory Domain Controller
- DNS Forwarders - 192.168.1.10
- Domain - au-team.irpo
- Password - P@ssw0rd
- Apply
- Запускаем и дожидаемся состояния - OK.

### BR-SRV:
**Перепускаем систему и проверяем**:
```bash
ping ya.ru
ping br-srv.au-team.irpo
ping hq-rtr.au-team.irpo
```
> Если все работает - то ОК!

### HQ-CLI:
**Перепроверяем 192.168.3.10:8081**
- Заходим в Domain
> Если сервер показывает статус OK, то идем дальше.

**От рута выполняем:**
```bash
nmcli con modify CLI-NET \
	ipv4.method auto \
	ipv4.ignore-auto-dns yes \
	ipv4.dns 192.168.3.10
```
```bash
nmcli con down CLI-NET
nmcli con up CLI-NET
```
**Открываем снова GUI и запускаем терминал, там прописываем**:
```bash
acc
```
- Пароль toor
- Выбрать Auth в Networking
- Прописать в Domain - au-team.irpo
- Прописать в Workgroup - au-team
- Apply
- Login: Administrator, Password: P@ssw0rd
> Если вход в домен произошел, то - ОК!

### BR-SRV
```bash
samba-tool group add hq
for i in $(seq 1 5); do samba-tool user add user$i.hq 'P@ssw0rd'; done
for i in $(seq 1 5); do samba-tool group addmembers hq user$i.hq; done
```
Проверим наличие группы hq в Samba, и созданных пользователей:
```bash
samba-tool group list
samba-tool group listmembers hq
```
```bash
admx-msi-setup
```
### HQ-CLI

**Перезапускаем и входим как Administrator**:
- Пароль: P@ssw0rd

**Открываем терминал**:
```bash
su -
toor
```
```bash
admx-msi-setup
```
```bash
roleadd hq wheel
```
```bash
rolelst
```
> **Проверяем наличие hq:wheel**
**Добавляем в sudoers данные строки:**
```bash
mcedit /etc/sudoers
, %AU-TEAM\\hq
Cmnd_Alias	SHELLCMD = /usr/bin/id, /bin/cat, /bin/grep
SHELLCMD
```
Для понимания где находятся эти строки, и куда их нужно добавить - пример того как это реализовано у меня:
```bash
User_Alias WHEEL_USERS = %wheel, %AU-TEAM\\hq # Первая строка
User_Alias XGRP_USERS = %xgrp
# User_Alias SUDO_USERS = %sudo

##
## Runas alias specification
##
Cmnd_Alias SHELLCMD = /usr/bin/id, /bin/cat, /bin/grep # Вторая строка
##
## User privilege specification
##
# root ALL=(ALL:ALL) ALL

## Uncomment to allow members of group wheel to execute any command
WHEEL_USERS ALL=(ALL:ALL) SHELLCMD # Третья строка
```
```
exit
```
**После того как вышли из под рута, выполняем kinit:**
```bash
kinit
P@ssw0rd
```
```bash
admc
```
**Настроим групповую политку:**
- Group Policy Objects
- au-team.irpo > правой кнопкой мыши
- Create a GPO and link to ths GPU
- Название: sudoers
- Ставим галочку в поле enforced
- Правой кнопкой мыши по sudoers > edit
- Machine
- Administative Templates
- Samba
- Unix Settings
- Sudo rights
- Enabled
- /usr/bin/id
- /bin/cat
- /bin/grep
- Применяем и выходим из admc.
```bash
gpupdate -f # Прописываем 2 раза подряд чтобы команда точно применилась, иногда не срабатывает с 1 раза.
```
**Заходим из под user5.hq:**
- Пароль - P@ssw0rd

```bash
sudo id
P@ssw0rd
sudo cat /root/.bashrc
sudo cat /root/.bashrc | grep root
```
> Если все команды выполняются, значит все выполнено верно.

## 📋 Задание 11: Удобным способом установите приложение Яндекс Браузер на HQ-CLI.

**Задание 11:**
- Установку браузера отметьте в отчёте.

> Готовый отчет можно взять - [тут.](./report_2026.odt)

**Самый лучший способ: установить напрямую через пакетный менеджер.**
```bash
apt-get update && apt-get install yandex-browser -y
```
## 📋 Задание 2: Сконфигурируйте файловое хранилище на сервере HQ-SRV + Задание 3: Настройте сервер сетевой файловой системы (nfs) на HQ-SRV.

**Задание 2:**

- При помощи двух подключенных к серверу дополнительных дисков размером 1 Гб сконфигурируйте дисковый массив уровня 0
- Имя устройства – md0, при необходимости конфигурация массива размещается в файле /etc/mdadm.conf
- Создайте раздел, отформатируйте раздел, в качестве файловой системы используйте ext4
- Обеспечьте автоматическое монтирование в папку /raid

**Задание 3:**
- В качестве папки общего доступа выберите /raid/nfs, доступ для чтения и записи исключительно для сети в сторону HQ-CLI
- На HQ-CLI настройте автомонтирование в папку /mnt/nfs
- Основные параметры сервера отметьте в отчёте

> Готовый отчет можно взять - [тут.](./report_2026.odt)

### HQ-SRV
```bash
lsblk
mdadm -C /dev/md0 -l 0 -n 2 /dev/sd{b,c}
lsblk
mkfs.ext4 /dev/md0
echo DEVICE partitions >> /etc/mdadm.conf
mdadm --detail --scan >> /etc/mdadm.conf
mkdir /raid
```
```bash
mcedit /etc/fstab
/dev/md0	/raid ext4 defaults	0	0
```
```bash
mount -a
df -h
lsblk
```
```bash
apt-get update && apt-get install -y nfs-{server,utils}
mkdir /raid/nfs
chmod 766 /raid/nfs
```
Комментируем первую строку в файле /etc/exports, в самом низу прописываем, то что идет ниже.
```
mcedit /etc/exports
/raid0/nfs 192.168.2.0/28(rw,no_subtree_check,no_root_squash)
```
**Применяем изменения:**
```bash
exportfs -arv
systemctl enable --now nfs-server.service
systemctl restart nfs-server.service
```
### HQ-CLI
```bash
apt-get update && apt-get install -y nfs-{server,utils}
mkdir /mnt/nfs
chmod 777 /mnt/nfs
```
```bash
mcedit /etc/fstab
192.168.1.10:/raid/nfs	/mnt/nfs	nfs	defaults	0	0
systemctl enable --now nfs-server.service
systemctl restart nfs-server.service
```
**Монтируем файловую систему и делаем финальную проверку RAID:**
```bash
mount -a
df -h
```
**Вывод команды:**
```bash
Файловая система       Размер Использовано  Дост Использовано% Cмонтировано в
udevfs                   5,0M         100K  5,0M            2% /dev
runfs                    1,3G         1,3M  1,3G            1% /run
/dev/sda2                 15G         7,1G  6,5G           53% /
tmpfs                    1,3G            0  1,3G            0% /dev/shm
tmpfs                    1,3G         4,0K  1,3G            1% /tmp
/dev/sda1                473M          55M  390M           13% /var/log
192.168.1.10:/raid/nfs   2,0G            0  1,9G            0% /mnt/nfs
tmpfs                    247M          56K  247M            1% /run/user/0
tmpfs                    247M          80K  247M            1% /run/user/812001105
```
> ⚠️ 💡 **Примечание**: Пробуем перезагрузку обоих машин и проверяем вывод через df -h снова, расшаренная файловая система с RAID - должна быть автоматически доступна.

## 📋 Задание 4: Настройте службу сетевого времени на базе сервиса chrony на маршрутизаторе ISP.

**Задание 4:**
- Вышестоящий сервер ntp на маршрутизаторе ISP - на выбор участника.
- Стратум сервера - 5
- В качестве клиентов ntp настройте: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV.

### ISP
```bash
apt-get update && apt-get install -y chrony tzdata
```
```bash
vim /etc/chrony.conf
initstepslew 10 ntp0.ntp-servers.net
pool 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0
```
**Запустим службу времени:**
```bash
systemctl restart chronyd
systemctl enable --now chronyd
timedatectl set-timezone Asia/Novosibirsk
```
**В качестве клиентов настроим: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV, выполнить настройку нужно идентично нижней на всех 4-ех клиентах.**
```bash
apt-get update && apt-get install -y chrony tzdata
vim /etc/chrony.conf
pool 172.16.1.1 iburst prefer
systemctl restart chronyd
systemctl enable --now chronyd
timedatectl set-timezone Asia/Novosibirsk
```
### HQ-RTR
**Хоть на HQ-RTR и не настраивается chrony, но часовой пояс укажем и там тоже.**
```bash
timedatectl set-timezone Asia/Novosibirsk
```
> ⚠️ 💡 **Примечание**: На HQ-CLI уже будет сервер времени, нужно лишь добавить новый pool.

## 📋 Задание 5:  Сконфигурируйте ansible на сервере BR-SRV

**Задание 5:**
- Сформируйте файл инвентаря, в инвентарь должны входить HQ-SRV, HQ-CLI, HQ-RTR и BR-RTR
- Рабочий каталог ansible должен располагаться в /etc/ansible
- Все указанные машины должны без предупреждений и ошибок отвечать pong на команду ping в ansible посланную с BR-SRV

### BR-SRV
```bash
apt-get update && apt-get install openssh-server ansible sshpass nano -y
```
```bash
nano /etc/ansible/hosts
[Alt]
hq-rtr.au-team.irpo ansible_ssh_user=net_admin ansible_ssh_pass=P@ssw0rd
hq-srv.au-team.irpo ansible_ssh_user=sshuser ansible_ssh_pass=P@ssw0rd
hq-cli.au-team.irpo ansible_ssh_user=sysadmin ansible_ssh_pass=P@ssw0rd
br-rtr.au-team.irpo ansible_ssh_user=net_admin ansible_ssh_pass=P@ssw0rd

[Alt:vars]
ansible_port=2026
```
```bash
nano /etc/ansible/ansible.cfg
[defaults]

interpreter_python = /usr/bin/python3

# some basic default values...
# uncomment this to disable SSH key host checking
host_key_checking = False
```
```bash
ansible -m ping all
```
> Если есть положительные ответы уже на этом моменте, то там где они положительные - настройку не производим.

> В моем случае HQ-SRV уже настроен на порт 2026 с данными для авторизации что я указал выше.
### HQ-SRV
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 2026
MaxAuthTries 2
AllowUsers sshuser
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```

### HQ-RTR и BR-RTR
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 2026
MaxAuthTries 2
AllowUsers net_admin
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```

### HQ-CLI 
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
useradd sysadmin
passwd sysadmin
P@ssw0rd
usermod -a -G remote sysadmin
```
```bash
vim /etc/openssh/sshd_config
Port 2026
MaxAuthTries 2
AllowGroups wheel remote
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```

### BR-SRV
```bash
ansible -m ping all
```
**Вывод команды**:
```bash
hq-srv.au-team.irpo | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
br-rtr.au-team.irpo | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
hq-rtr.au-team.irpo | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
hq-cli.au-team.irpo | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
> ⚠️ 💡 **Важно**: Проверяем чтобы вывод команды совпадал, если все совпдает, значит задание выполнено верно.

## 📋 Задание 6:  Разверните веб приложение в docker на сервере BR-SRV.

**Задание 6:**
- Средствами docker должен создаваться стек контейнеров с веб приложением и базой данных
- Используйте образы site_latest и mariadb_latest располагающиеся в директории docker в образе Additional.iso
- Основной контейнер testapp должен называться tespapp
- Контейнер с базой данных должен называться db
- Импортируйте образы в docker, укажите в yaml файле параметры подключения к СУБД, имя БД - testdb, пользователь testс паролем P@ssw0rd, порт приложения 8080, при необходимости другие параметры
- Приложение должно быть доступно для внешних подключений через порт 8080

### BR-SRV
```bash
apt-get update && apt-get install docker-ce docker-compose -y
systemctl enable --now docker.socket docker.service
systemctl restart docker.socket docker.service
```
```bash
mount /dev/sr0 /mnt
```
**Проверяем точку монтирования:**
```bash
lsblk
```
**Сверяем вывод:**
```bash
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0    10G  0 disk 
├─sda1   8:1    0   503M  0 part /var/log
└─sda2   8:2    0   9.5G  0 part /
sr0     11:0    1 929.7M  0 rom  /mnt
```
**Проверяем содержимое внутри образа:**
```bash
ls -la /mnt
```
```bash
total 38
dr-xr-xr-x  1 root root   256 Nov 23  2019 .
drwxr-xr-x 24 root root  4096 Dec 11 01:08 ..
dr-xr-xr-x  1 root root   332 Nov 23  2019 docker
dr-xr-xr-x  1 root root   150 Nov 23  2019 playbook
-r-xr-xr-x  1 root root 32527 Oct 13 04:22 Users.csv
dr-xr-xr-x  1 root root   220 Nov 23  2019 web
```
**Копируем docker с образа на систему:**
```bash
cp -r /mnt/docker /root/docker
```
**Проверяем файлы:**
```bash
ls -la /root/docker
```
```bash
total 951964
dr-xr-xr-x 2 root root      4096 Dec 12 22:54 .
drwx------ 8 root root      4096 Dec 12 22:54 ..
-r-xr-xr-x 1 root root 333014016 Dec 12 22:54 mariadb_latest.tar
-r-xr-xr-x 1 root root 282003968 Dec 12 22:54 postgresql_latest.tar
-r-xr-xr-x 1 root root      2716 Dec 12 22:54 readme.txt
-r-xr-xr-x 1 root root 359760896 Dec 12 22:54 site_latest.tar
```
**Загружаем в docker**
```bash
docker load -i /root/docker/site_latest.tar
docker load -i /root/docker/mariadb_latest.tar
```
**Сверяем и проверяем TAG, это потребуется для image: в docker-compose.yaml**
```bash
docker image ls
```
**У меня получился такой вывод:**
```bash
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
site         latest    015b4b821098   2 months ago   353MB
mariadb      10.11     bc52d24721da   4 months ago   327MB
```
**Создаем директорию и файл конфигурации:**
```bash
mkdir testapp
nano testapp/docker-compose.yaml
```
```bash
services:
  testapp:
    image: site:latest
    container_name: testapp # В задании указано tespapp, скорее всего опечатка составителей.
    restart: always
    depends_on:
      - db
    ports:
      - "8080:8000"
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_PORT: 3306
      DB_USER: testc
      DB_PASS: P@ssw0rd   

  db:
    image: mariadb:10.11
    container_name: db
    restart: always
    environment:
      MARIADB_NAME: testdb
      MARIADB_USER: testc
      MARIADB_PASS: P@ssw0rd   
      MARIADB_ROOT_PASSWORD: toor
    volumes:
      - /root/testapp/db_data:/var/lib/mysql
```
**Переходим в директорию если ещё не перешли, и поднимаем Docker:**
```bash
cd testapp/
docker compose up -d
docker ps
```
**Сверяем вывод:**
```bash
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS                         PORTS      NAMES
2f8db625ddfc   site:latest     "sh -c 'python3 -m a…"   22 seconds ago   Restarting (1) 2 seconds ago              testapp
13af1bc1529e   mariadb:10.11   "docker-entrypoint.s…"   22 seconds ago   Up 21 seconds                  3306/tcp   db
```
> [!WARNING]
> На этом этапе заходим на HQ-CLI, и через Firefox пробуем зайти на 192.168.3.10:8080, если страница не открывается, то выполняем действия ниже, если открылась, задание выполнено.
### BR-SRV
```bash
docker exec -it db /bin/bash
mariadb -u root -p
toor
```
```bash
SHOW DATABASES;
```
**Причина кроется тут, нет нужной базы данных, пропишем ее и пользователя, а так же привилегии в ручную:**
```bash
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.001 sec)
```
**Выполняем по очереди, в точности как у меня:**
```bash
CREATE DATABASE testdb;
MariaDB [(none)]> CREATE USER 'testc'@'%' IDENTIFIED BY 'P@ssw0rd';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON testdb.* TO 'testc'@'%';
FLUSH PRIVILEGES;
```
> [!NOTE]  
> Пробуем снова с HQ-CLI зайти на 192.168.3.10:8080, если страница открывается, задание выполнено.
