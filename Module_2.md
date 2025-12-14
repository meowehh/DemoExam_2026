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
> [!TIP]
> Если все работает - то ОК!

### HQ-CLI:
**Перепроверяем 192.168.3.10:8081**
- Заходим в Domain
> [!TIP]
> Если сервер показывает статус OK, то идем дальше.

**От рута выполняем:**
```bash
nmcli con modify DHCP-CLI \
	ipv4.method auto \
	ipv4.ignore-auto-dns yes \
	ipv4.dns 192.168.3.10
```
```bash
nmcli con down DHCP-CLI
nmcli con up DHCP-CLI
```
**Открываем снова GUI и запускаем терминал, там прописываем**:
```bash
acc
```
- Пароль toor
- Выбрать Аутентификация в разделе пользователи. (Если интерфейс на английском языке, то Auth в Networking)
- Обязательно выибраем - Домен Active Directory
- Прописать в домен - au-team.irpo
- Прописать в рабочую группу - au-team
- Имя компьютера - hq-cli
- Выбираем обязательно SSID, а не Winbind.
- Сохранить
- Логин: Administrator, Пароль: P@ssw0rd
- Включить групповые политики!
- Напротив kerberos галочку НЕ ставить.
- Применяем изменения.
> [!TIP]
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
> [!IMPORTANT]
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
gpupdate -f
```
> [!IMPORTANT]
> Прописываем 2 раза подряд чтобы команда точно применилась, иногда не срабатывает с 1 раза.

**Заходим из под user5.hq:**
- Пароль - P@ssw0rd

```bash
sudo id
P@ssw0rd
sudo cat /root/.bashrc
sudo cat /root/.bashrc | grep root
```
> [!TIP]
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
  
> [!IMPORTANT]
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
> [!WARNING]
> Комментируем первую строку в файле /etc/exports, в самом низу прописываем, то что идет ниже.
```
mcedit /etc/exports
/raid/nfs 192.168.2.0/28(rw,no_subtree_check,no_root_squash)
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
> [!IMPORTANT]
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
> [!NOTE]
> ⚠️ 💡 **Примечание**: На HQ-CLI уже будет сервер времени, нужно лишь добавить новый pool и перезагрузить chronyd.

### ISP
```bash
systemctl restart chronyd
```
### HQ-SRV, HQ-CLI, BR-RTR, BR-SRV
```bash
chronyc sources
```
**Пример вывода:**
```bash
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* 172.16.1.1                    5   6    77    35   +859ns[  -23us] +/-  342us
```
> [!NOTE]
> ⚠️ 💡 **Примечание**: На HQ-CLI будет 2 сервера в выводе, но приоритетный ISP.

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
> [!TIP]
> Если есть положительные ответы уже на этом моменте, то там где они положительные - настройку не производим.

> [!NOTE]
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
> [!TIP]
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
**Либо может быть так:**
```bash
CONTAINER ID   IMAGE           COMMAND                  CREATED       STATUS             PORTS                                       NAMES
264276eed58b   site:latest     "sh -c 'python3 -m a…"   27 seconds ago   Up 26 seconds   0.0.0.0:8080->8000/tcp, :::8080->8000/tcp   testapp
8e79d22f6fa7   mariadb:10.11   "docker-entrypoint.s…"   27 seconds ago   Up 26 seconds   3306/tcp                                    db
```
> [!CAUTION]
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
> [!Tip] 
> Пробуем снова с HQ-CLI зайти на 192.168.3.10:8080, если страница открывается, задание выполнено.

## 📋 Задание 7:  Разверните веб приложение на сервере HQ-SRV.

**Задание 7:**

- Используйте веб-сервер apache
- В качестве системы управления базами данных используйте mariadb
- Файлы веб приложения и дамп базы данных находятся в директории web образа Additional.iso
- Выполните импорт схемы и данных из файла dump.sql в базу данных webdb
- Создайте пользователя webс паролем P@ssw0rd и предоставьте ему права доступа к этой базе данных
- Файлы index.php и директорию images скопируйте в каталог веб сервера apache
- В файле index.php укажите правильные учётные данные для подключения к БД
- Запустите веб сервер и убедитесь в работоспособности приложения
- Основные параметры отметьте в отчёте

> [!IMPORTANT]
> Готовый отчет можно взять - [тут.](./report_2026.odt)

### HQ-SRV
```bash
apt-get update && apt-get install apache2 mariadb php8.2 apache2-mod_php8.2 php8.2-mysqli -y
```
```bash
systemctl enable --now httpd2 mariadb
systemctl restart httpd2 mariadb
```
```bash
mkdir /mnt/add_cd
mount /dev/sr0 /mnt/add_cd
ls -la /mnt/add_cd
```
**Сверяем вывод:**
```bash
total 38
dr-xr-xr-x 1 root root   256 Nov 23  2019 .
drwxr-xr-x 3 root root  4096 Dec 13 12:54 ..
dr-xr-xr-x 1 root root   332 Nov 23  2019 docker
dr-xr-xr-x 1 root root   150 Nov 23  2019 playbook
-r-xr-xr-x 1 root root 32527 Oct 13 04:22 Users.csv
dr-xr-xr-x 1 root root   220 Nov 23  2019 web
```
```bash
cp -r /mnt/add_cd/web /root/web
ls -la /root/web
```
**Сверяем вывод:**
```bash
total 36
dr-xr-xr-x 2 root root  4096 Dec 13 12:55 .
drwx------ 9 root root  4096 Dec 13 12:55 ..
-r-xr-xr-x 1 root root   415 Dec 13 12:55 dump.sql
-r-xr-xr-x 1 root root  3964 Dec 13 12:55 index.php
-r-xr-xr-x 1 root root 16780 Dec 13 12:55 logo.png
```
**Создаем базу данных webdb**
```bash
mysql -u root -e "CREATE DATABASE webdb;"
```
**Импортируем дамп**
```bash
mysql -u root webdb < /root/web/dump.sql
```
**Проверяем импорт**
```bash
mysql -u root -e "USE webdb; SHOW TABLES;"
```
**Сверяем вывод:**
```bash
+-----------------+
| Tables_in_webdb |
+-----------------+
| employees       |
+-----------------+
```
**Создаем пользователя 'web' с паролем 'P@ssw0rd'**
```bash
mysql -u root -e "CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
```
**Даем права на базу webdb**
```bash
mysql -u root -e "GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';"
```
**Применяем изменения**
```bash
mysql -u root -e "FLUSH PRIVILEGES;"
```
**Проверяем создание пользователя**
```bash
mysql -u root -e "SELECT user, host FROM mysql.user;"
```
**Сверяем вывод:**
```bash
+-------------+---------------------+
| User        | Host                |
+-------------+---------------------+
| root        | 127.0.0.1           |
| root        | ::1                 |
| root        | hq-srv.au-team.irpo |
| mariadb.sys | localhost           |
| root        | localhost           |
| web         | localhost           |
+-------------+---------------------+
```
```bash
cp /root/web/index.php /var/www/html/
cp /root/web/logo.png /var/www/html/
```
```bash
chown -R apache2:webmaster /var/www/html/
chmod 755 /var/www/html/
```
```bash
ls -la /var/www/html/index.php
ls -la /var/www/html/logo.png
```
```bash
vim /var/www/html/index.php
```
**Приводим к такому виду:**
```bash
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
```
```bash
mv /var/www/html/index.html /var/www/html/index.html.default
ls -la /var/www/html/
```
**Сверяем вывод:**
```bash
drwxr-sr-x 4 apache2 webmaster  4096 Dec 14 14:23 .
drwxr-xr-x 9 root    webmaster  4096 Dec 13 12:20 ..
drw-r-Sr-- 2 apache2 webmaster  4096 Oct 12  2010 addon-modules
-rw-r--r-- 1 apache2 webmaster    45 Jul 28 15:35 index.html.default
-rw-r--r-- 1 apache2 webmaster  3968 Dec 13 18:58 index.php
-r-xr-xr-x 1 root    webmaster 16780 Dec 14 14:23 logo.png
```
```bash
systemctl restart httpd2 mariadb
```
> [!NOTE]
> Заходим на HQ-CLI, проверяем 192.168.1.10, должен открыться сайт, фио - DemoTest, отдел - DemoTest. Если эти поля уже заполены, значит все выполнено верно.

> [!TIP]
> Задание выполенено, веб-приложение работает на Apache + PHP + MariaDB, доступно по сети, выполняет CRUD-операции с базой данных сотрудников.

## 📋 Задание 8: На маршрутизаторах сконфигурируйте статическую трансляцию портов.

**Задание 8:**
 
- Пробросьте порт 8080 в порт приложения testapp BR-SRV на маршрутизаторе BR-RTR, для обеспечения работы приложения testapp извне.
- Пробросьте порт 8080 в порт веб приложения на HQ-SRV на маршрутизаторе HQ-RTR, для обеспечения работы веб приложения извне.
- Пробросьте порт 2026 на маршрутизаторе HQ-RTR в порт 2026 сервера HQ-SRV, для подключения к серверу по протоколу ssh из внешних сетей.
- Пробросьте порт 2026 на маршрутизаторе BR-RTR в порт 2026 сервера BR-SRV, для подключения к серверу по протоколу ssh из внешних сетей.

### BR-RTR
```bash
apt-get update && apt-get install iptables -y
```
**Проброс порта 8080 для testapp (Docker приложение на BR-SRV).**
```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.3.10:8080
iptables -A FORWARD -p tcp -d 192.168.3.10 --dport 8080 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
**Проброс порта 2026 для SSH.**
```bash
iptables -t nat -A PREROUTING -p tcp --dport 2026 -j DNAT --to-destination 192.168.3.10:2026
iptables -A FORWARD -p tcp -d 192.168.3.10 --dport 2026 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
**Сохранение правил и перезапуск.**
```bash
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
systemctl enable --now iptables
```
**Выполним проверку:**
```bash
iptables -t nat -L -n -v
```
**Сверяем вывод:**
```bash
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.3.10:8080
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:2026 to:192.168.3.10:2026

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination 
```

### HQ-RTR
```bash
apt-get update && apt-get install iptables -y
```
**Проброс порта 8080 для веб-приложения Apache (перенаправляем на порт 80 HQ-SRV).**
```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
**Проброс порта 2026 для SSH.**
```bash
iptables -t nat -A PREROUTING -p tcp --dport 2026 -j DNAT --to-destination 192.168.1.10:2026
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 2026 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
**Сохранение правил и перезапуск.**
```bash
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
systemctl enable --now iptables
```
**Выполним проверку:**
```bash
iptables -t nat -L -n -v
```
**Сверяем вывод:**
```bash
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.1.10:80
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:2026 to:192.168.1.10:2026

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination 
```
> [!NOTE]
> Если выводы совпадают, задание выполнено верно.

## 📋 Задание 8: Настройте веб-сервер nginx как обратный прокси-сервер на ISP.

**Задание 8**:

- При обращении по доменному имени web.au-team.irpo у клиента должно открываться веб приложение на HQ-SRV.
- При обращении по доменному имени docker.au-team.irpo клиента должно открываться веб приложение testapp.

### ISP
```bash
apt-get update && apt-get install nginx nano -y
```
```bash
cp /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-available.d/web.conf
nano /etc/nginx/sites-available.d/web.conf
```
```bash
upstream web.au-team.irpo {
        server 172.16.1.10:80;
}
server {
        listen  8080;
        server_name _;

        location / {
                # root /var/www/html;

                # autoindex off;
                # autoindex_exact_size on;
                # autoindex_localtime off;

                # expires off;

                # cooperate with mod_realip in apache-1.3 or mod_rpaf in apache-2.x
                #       proxy_redirect off;
                #       proxy_set_header Host $host;
                #       proxy_set_header X-Real-IP $remote_addr;
                #       proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_pass http://web.au-team.irpo;
                #
                # NB: it's better for URI canonicalization that apache sits on :80
                # (even if that's only 127.0.0.1:80)
                #
                # see also set_real_ip_from, real_ip_header if this nginx
                # would need to cooperate with another one acting as a frontend
        }

#               charset         on;
#               source_charset  koi8-r;

#               access_log  /var/log/nginx/access.log;
}
```
```bash
cp /etc/nginx/sites-available.d/web.conf /etc/nginx/sites-available.d/docker.conf
nano /etc/nginx/sites-available.d/docker.conf
```
```bash
upstream web.au-team.irpo {
        server 172.16.2.10
}
server {
        listen  8080;
        server_name _;

        location / {
                # root /var/www/html;

                # autoindex off;
                # autoindex_exact_size on;
                # autoindex_localtime off;

                # expires off;

                # cooperate with mod_realip in apache-1.3 or mod_rpaf in apache-2.x
                #       proxy_redirect off;
                #       proxy_set_header Host $host;
                #       proxy_set_header X-Real-IP $remote_addr;
                #       proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_pass http://docker.au-team.irpo:8080;
                #
                # NB: it's better for URI canonicalization that apache sits on :80
                # (even if that's only 127.0.0.1:80)
                #
                # see also set_real_ip_from, real_ip_header if this nginx
                # would need to cooperate with another one acting as a frontend
        }

#               charset         on;
#               source_charset  koi8-r;

#               access_log  /var/log/nginx/access.log;
}
```
```bash
ln -s /etc/nginx/sites-available.d/web.conf /etc/nginx/sites-enabled.d/web.conf
ln -s /etc/nginx/sites-available.d/docker.conf /etc/nginx/sites-enabled.d/docker.conf
```
```bash
systemctl restart nginx
systemctl reload nginx
systemctl enable --now nginx
```
