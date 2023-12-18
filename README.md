ДЕМО экзамен
===

Старая картинка, нужна лишь для примера
![](http://188.120.246.191:8098/uploads/2c612c30-05f1-42ef-901b-9a591eba9ad4.png)
![](https://user-images.githubusercontent.com/28905300/175996163-e7dbdb7b-2d18-4150-906a-ac925f3a66c7.png)

## Первоначальная настройка

Использовать команду
```bash
sudo apt-get update -y
```

### Если sudo не работает

```bash
su
apt-get install sudo -y
sudo usermod -aG sudo user # user - пользователь через которого идет настройка
su user
```

### Изменение hostname

В /etc/hostname удалить старое название и добавить новое

```bash
sudo nano /etc/hostname

[новое название]
```

В /etc/hosts изменить старое название на новое в строчке с 127.0.0.1

```bash
sudo nano /etc/hosts

127.0.0.1 localhost
127.0.1.1 [новое название]
```

### Изменение/добавление ip для портов
В /etc/network/interfaces нужно прописать новые порты(ens37, ens38, ...)
```bash
sudo nano /etc/network/interfaces
```
Пример:
```bash
auto ens37 # ens37 - настраиваемый порт
iface ens37 inet static
address 3.3.3.1 # ip порта
gateway 3.3.3.1 # ip роутера, к которому подключен порт
netmask 255.255.255.0 # маска
```

#### Все интрефейсы
###  ИНТЕРФЕЙСЫ МОГУТ ОТЛИЧАТЬСЯ ОТ ВАШИХ, ПЕРЕПРОВЕРЯЙТЕ С MAC-адресом (`ip a`). Отсчет ens может начинаться не с 36, а с 37
##### ISP
```bash
auto ens36
iface ens36 inet static
address 3.3.3.1
netmask 255.255.255.0
gateway 3.3.3.1

auto ens37
iface ens37 inet static
address 7.7.7.1
netmask 255.255.255.0
gateway 7.7.7.1

auto ens38
iface ens38 inet static
address 8.8.8.1
netmask 255.255.255.0
gateway 8.8.8.1
```

##### RTR-L
```bash
auto ens36
iface ens36 inet static
address 7.7.7.100
netmask 255.255.255.0
gateway 7.7.7.1

auto ens37
iface ens37 inet static
address 192.168.100.254
netmask 255.255.255.0
gateway 192.168.100.254
```

##### RTR-R
```bash
auto ens36
iface ens36 inet static
address 8.8.8.100
netmask 255.255.255.0
gateway 8.8.8.1

auto ens37
iface ens37 inet static
address 172.16.100.254
netmask 255.255.255.0
gateway 172.16.100.254
```

##### WEB-R
```bash
auto ens36
iface ens36 inet static
address 172.16.100.100
netmask 255.255.255.0
gateway 172.16.100.254
```

##### WEB-L
```bash
auto ens36
iface ens36 inet static
address 192.168.100.100
netmask 255.255.255.0
gateway 192.168.100.254
dns-nameservers 192.168.100.200
```
Применить изменения без перезагрузки
```bash
sudo systemctl restart networking 
# ПРИ ИСПОЛЬЗОВАНИИ ЭТОЙ КОМАНДЫ МОЖЕТ ОТВАЛИТЬСЯ SSH И ИНТЕРНЕТ
```
Лучше использовать
```bash
sudo reboot
```

## IP forwarding
### ISP, RTR-L, RTR-R
Раскоментить строчку net.ipv4.ip_forward=1 в sudo nano /etc/sysctl.conf 

![](http://188.120.246.191:8098/uploads/3785ac62-b0e3-4ec3-81da-6a2565bd83f9.png)

### Установить iptables для RTR-L и RTR-R
```bash
sudo apt-get install iptables -y
```


## Tunnels

### RTR-L, RTR-R
В файле /etc/modules добавить
```bash
ip_gre
```
Прописать команду:
```bash
sudo modprobe ip_gre
```
В /etc/network/interfaces добавить туннели
#### RTR-L
```bash
auto gre30
  iface gre30 inet tunnel
  address 172.16.1.1
  netmask 255.255.255.0
  mode gre
  local 7.7.7.100 # ip RTR-L - ISP
  endpoint 8.8.8.100 # ip RTR-R - ISP
  ttl 255
  post-up ip route add 172.16.100.0/24  via 172.16.1.2
# 172.16.100.0/24 - сеть RTR-R - WEB-R
```
#### RTR-R
```bash
auto gre30
  iface gre30 inet tunnel
  address 172.16.1.2
  netmask 255.255.255.0
  mode gre
  local 8.8.8.100
  endpoint 7.7.7.100
  ttl 255
  post-up ip route add 192.168.100.0/24 via 172.16.1.1
# 192.168.100.0/24 - сеть RTR-L - WEB-L
```
На RTR-R и RTR-L
```bash
sudo iptables -A INPUT -p gre -j ACCEPT
```
```bash
sudo reboot
```
## Firewall
### ISP, RTR-R, RTR-L
На ISP устанавливать в последнюю очередь, либо отключить его, если уже установили и интеренет не работает на других компах `sudo systemctl stop firewalld.service`

Установить firewall:
```bash

sudo apt-get install firewalld

```

### ISP

```bash

sudo firewall-cmd --permanent --zone=trusted --add-interface=ens33

```

### RTR-R
```bash

sudo firewall-cmd --permanent --zone=external --add-interface=ens36 
#ens36 - интерфейс с ip 8.8.8.100

sudo firewall-cmd --permanent --zone=external --add-interface=gre30
#gre30 - порт с тунелем

sudo firewall-cmd --reload

```

### RTR-L
```bash

sudo firewall-cmd --permanent --zone=external --add-interface=ens36 
#ens36 - интерфейс с ip 7.7.7.100

sudo firewall-cmd --permanent --zone=external --add-interface=gre30
#gre30 - порт с тунелем

sudo firewall-cmd --reload

```

### ISP

```bash

sudo firewall-cmd --permanent --zone=public --add-forward

sudo firewall-cmd --permanent --zone=external --add-forward

sudo firewall-cmd --permanent --zone=public --add-interface=ens36

sudo firewall-cmd --permanent --zone=public --add-interface=ens37

sudo firewall-cmd --permanent --zone=public --add-interface=ens38

sudo firewall-cmd --reload

```


## OSPF
### RTR-R, RTR-L
```bash

sudo apt-get install frr -y

```

В файле ```/etc/frr/daemons``` изменить ```ospfd=no``` на ```ospfd=yes```

```bash

sudo nano /etc/frr/daemons


osfpd=yes

```


### ISP, RTR-R, RTR-L

```bash

sudo firewall-cmd --permanent --zone=external --add-protocol=89

sudo firewall-cmd --permanent --zone=external --add-protocol=ospf

sudo firewall-cmd --reload

```

### RTR-R, RTR-L
```bash

sudo systemctl restart frr

sudo systemctl enable frr --now

```

```bash

sudo vtysh

conf t
router ospf
ospf router-id 172.16.1.1 
#id зависит от роутера: RTR-R 172.16.1.2 | RTR-L 172.16.1.1

network 172.16.1.0/24 area 0 #НЕ МЕНЯЕТСЯ
network 192.168.100.0/24 area 0 #МЕНЯЕТСЯ
# network зависит от роутера: RTR-R 172.16.100.0/24 | RTR-L 192.168.100.0/24

passive-interface ens36
passive-interface ens37

do w
end
exit

```

### RTR-L
```bash

sudo firewall-cmd --permanent --zone=external --add-port=53/udp

sudo firewall-cmd --permanent --zone=external --add-port=80/tcp

sudo firewall-cmd --permanent --zone=external --add-port=443/tcp

sudo firewall-cmd --permanent --zone=trusted --add-interface=ens37
#ens - порт с ip 192.168.100.0

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/0 forward-port port=443 protocol=tcp to-port=443 to-addr=192.168.100.100'

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/0 forward-port port=80 protocol=tcp to-port=80 to-addr=192.168.100.100'

--- НЕ ДЕЛАТЬ ---
sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0 forward-port port=2222 protocol=tcp to-port to-addr=192.168.100.100'
-----------------

sudo firewall-cmd --reload

```

### RTR-R
```bash

sudo firewall-cmd --permanent --zone=external --add-port=80/tcp --add-port=443/tcp

sudo firewall-cmd --permanent --zone=trusted --add-interface=ens37
#ens - порт с ip 172.16.100.0

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/0 forward-port port=443 protocol=tcp to-port=443 to-addr=172.16.100.100'

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/0 forward-port port=80 protocol=tcp to-port=80 to-addr=172.16.100.100'

--- НЕ ДЕЛАТЬ ---
sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0 forward-port port=2222 protocol=tcp to-port to-addr=172.16.100.100'
-----------------

sudo firewall-cmd --reload

```



## DNS
### ISP

Установить bind9
```bash
sudo apt-get install bind9 -y
```
Создать папку /opt/dns
```bash
sudo mkdir /opt/dns
```
Скопировать БД
```bash
sudo cp /etc/bind/db.local /opt/dns/demo.db
```
Изменить владельца папки
```bash
sudo chown -R bind:bind /opt/dns
```
Добавить в файл `/etc/apparmor.d/usr.sbin.named`:
```bash
/opt/dns/** rw,
```
Пример:
![](http://188.120.246.191:8098/uploads/b3139d1c-e713-4694-ab17-786a917b3f8d.png)


Перезапустить сервис
```bash
sudo systemctl restart apparmor.service
```
В файле `/etc/bind/named.conf.options` добавить/изменить:
```bash
forwarders {7.7.7.100;};

dnssec-valitation no;

listen-on-v6 { none; };
allow-query { any; };
```
ПРИМЕР:
![](http://188.120.246.191:8098/uploads/60afc993-ecfb-4784-9b6c-965cdcad92ba.png)


В файл ```/etc/bind/named.conf.default-zones``` добавить в конец:
```bash
zone "demo.wsr" {
   type master;
   allow-transfer { any; };
   file "/opt/dns/demo.db";
};
```
Изменить файл ```/opt/dns/demo.db``` на:
```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
$ORIGIN demo.wsr.
@       IN      SOA     isp.demo.wsr. root.demo.wsr. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      isp.demo.wsr.
;
isp     IN      A       3.3.3.1
www     IN      A       7.7.7.100
www     IN      A       8.8.8.100
internet IN CNAME  isp.demo.wsr.
;
$ORIGIN int.demo.wsr.
@       IN      NS      srv.int.demo.wsr.
        IN      NS      isp.demo.wsr.
;
srv.int.demo.wsr. A     7.7.7.100
```
Старый пример:
![](http://188.120.246.191:8098/uploads/b6087eda-16a7-4c87-8d93-20c81c03e7a4.png)


### ISP
```bash

sudo firewall-cmd --permanent --zone=public --add-service=dns

sudo firewall-cmd --permanent --zone=public --add-port=53/udp

sudo firewall-cmd --reload

```

Перезапустить сервис bind9
```bash

sudo systemctl restart apparmor
sudo systemctl restart bind9

```


### RTR-L
```bash

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/24 forward-port port=53 protocol=udp to-port=53 to-addr=192.168.100.200'

sudo firewall-cmd --reload

```

### CLI
В настройках подключения изменить настройки как на картинке
![](http://188.120.246.191:8098/uploads/0b417163-0345-4af2-bec3-247c4b8993eb.png)
Проверить DNS
![](http://188.120.246.191:8098/uploads/a05d055b-cff6-4214-b773-af45d7a7ae5d.png)
Отключить интерфейс с интернет сетью, выключить firewall

### SRV

Изменить имя сервера
![](http://188.120.246.191:8098/uploads/3d6f3c1d-82c0-4ade-b400-b535905ecc22.png)
![](http://188.120.246.191:8098/uploads/e7783584-3d57-4276-a6b4-01f5ce10d81c.png)
![](http://188.120.246.191:8098/uploads/5ef97063-d8f5-49ec-a230-42bfb7f394f6.png)

Поменять ip
```bash

ip: 192.168.100.200
gateway: 192.168.100.254
DNS: 192.168.100.200

```

Добавить установить DNS
![](http://188.120.246.191:8098/uploads/8f04f56c-0c23-4e74-9cfd-3190d9569bcb.png)

![](http://188.120.246.191:8098/uploads/e3393556-e064-4ed2-9ff4-b5f1d79da4b1.png)

![](http://188.120.246.191:8098/uploads/3fdd3d77-6d4f-4b17-963a-691c6bd24d06.png)

### Настройка
![](http://188.120.246.191:8098/uploads/b2f3066e-f937-4114-97f8-8b363dc24824.png)
![](http://188.120.246.191:8098/uploads/cabdf703-9184-4a84-a535-51fa9f84b508.png)
![](http://188.120.246.191:8098/uploads/cb21e5a0-fdcf-4f35-b49e-9312c64c3e97.png)
![](http://188.120.246.191:8098/uploads/c0da6b32-9f16-40da-95b1-61bc41fa1945.png)
![](http://188.120.246.191:8098/uploads/74385a51-4786-410e-8b30-cab29d98cf9b.png)
![](http://188.120.246.191:8098/uploads/b8da9f04-c474-4983-94ad-a07eb4096281.png)

Открыть int.demo.wsr и в нем пкм и выбрать пункт
![](http://188.120.246.191:8098/uploads/d68f735e-e763-4e08-98cc-26dc21ce0fd3.png)

Заполнить по примеру(ip=192.168.100.100):
![](http://188.120.246.191:8098/uploads/4b542f33-d583-4613-bd55-9e5424c12eae.png)
Заполнить по таблице (Здесь IP старые):
![](http://188.120.246.191:8098/uploads/4547732a-1ebf-4ed5-9842-64ada3f3f4dc.png)
Результат:
![](http://188.120.246.191:8098/uploads/fd729982-1222-4d29-9ff4-795d1f67ffde.png)
Так же пкм
![](http://188.120.246.191:8098/uploads/dbd5d105-76cb-4301-a19d-92e6e8de96c0.png)
![](http://188.120.246.191:8098/uploads/0a47722c-22fa-4e76-bace-534526eb9d4d.png)
![](http://188.120.246.191:8098/uploads/4ccd74ad-86e6-4a83-b93c-fc3218cee268.png)
![](http://188.120.246.191:8098/uploads/8b8dd3cc-768f-4ceb-91bd-441a06a0f647.png)
И повторить тоже самое для dns,webapp-L и webapp-R
Результат (без '1'):
![](http://188.120.246.191:8098/uploads/28b81d92-ecb5-4623-98e5-0aab20a4f283.png)

После чего
![](http://188.120.246.191:8098/uploads/06a2de67-de6d-49fe-93d3-c2a6d44ba78c.png)

![](http://188.120.246.191:8098/uploads/86e97d06-8ffe-406d-9ad7-70709ddeb91f.png)
![](http://188.120.246.191:8098/uploads/c7f3eed0-9f57-4589-a5c5-b6771553bfaa.png)
![](http://188.120.246.191:8098/uploads/8df3b217-c5e9-4243-ab83-de9b582bc9ae.png)
![](http://188.120.246.191:8098/uploads/eec6e702-2c0b-4a6d-83ae-b18187604723.png)
![](http://188.120.246.191:8098/uploads/22893ae4-89ea-4794-8a45-bcc2d1d2ffd1.png)

## Синхронизация времени/Chrony:

### ISP
```bash

sudo apt-get install chrony -y

```
Открыть файл с настройки
```bash

sudo nano /etc/chrony/chrony.conf

```
Добавить в файл
```bash

local stratum 4
allow 7.7.7.0/24
allow 3.3.3.0/24

```
Результат:
![](http://188.120.246.191:8098/uploads/11c61243-0a01-4e95-b53a-967868f33bde.png)

Перезапустить chrony

```bash

sudo systemctl restart chronyd 

```
### SRV
В PowerShell ввести команды:
```bash

New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow

w32tm /query /status

Start-Service W32Time

w32tm /config /manualpeerlist:7.7.7.1 /syncfromflags:manual /reliable:yes /update

Restart-Service W32Time

```

### CLI
Запустить PowerShell от имени администратора
![](http://188.120.246.191:8098/uploads/c325765f-7307-4c96-8fa8-0e5cfd2eb068.png)
И ввести команы:
```bash 

New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow

Start-Service W32Time

Restart-Service W32Time

Set-Service -Name W32Time -StartupType Automatic

```
### Смена настроек интерфейсов

### RTR-L, RTR-R, WEB-R, WEB-L калаш мне в зад это армия тротил мне в жопу
Узнать ip для ens33
```bash

ip a

```

Изменить настройки для ens33 в ```/etc/network/interfaces```
```bash

allow-hotplug ens33
auto ens33
iface ens33 inet static
address <ip> #ip, который назначен в ip a для ens33
netmask 255.255.255.0

```
Изменить файл ```/etc/resolv.conf``` на
```bash

domain int.demo.wsr
search int.demo.wsr
nameserver 192.168.100.200

```




### WEB-R, WEB-L, RTR-R, RTR-L
Установить chrony:
```bash

sudo apt-get install chrony -y

```
Настроить конфиг:
```bash

nano /etc/chrony/chrony.conf

```

Изменить строчку:
```bash

pool 2.debian.pool.ntp.org iburst

```

На:
```bash

pool ntp.int.demo.wsr iburst

```

### ISP

В ```/etc/chrony/chrony.conf``` добавить:
```bash

server ISP
local stratum 4
allow 7.7.7.1
allow 7.7.7.100
allow 3.3.3.1
allow 3.3.3.10

```
И закоментить 

```bash

pool 2.debian.pool.ntp.org iburst

```
Пример:
![](http://188.120.246.191:8098/uploads/d9013727-fb06-4e36-87d4-791e5cdbbe34.png)


И перезапустить сервис:

```bash

sudo systemctl restart chronyd

```

### RTR-R, RTR-L

Разрешить в firewall ntp сервер:
```bash

sudo firewall-cmd --permanent --zone=external --add-service=ntp

sudo firewall-cmd --reload

```

## Файловый сервер
### SRV
#### Настройка дисков
Добавить в параметрах виртуальной машины 2 жестких диска по 2 гб
Нажать "Win+X" и выбрать "Disk Managent"
Нажать пкм по выключиным дискам и включить их
![](http://188.120.246.191:8098/uploads/a28c32ff-e533-4cd6-a2e8-443b9afa1302.png)
Также пкм и инициализировать их
![](http://188.120.246.191:8098/uploads/7f0eb302-0f1d-4ef0-acd6-57c23e35f086.png)
![](http://188.120.246.191:8098/uploads/226ce8b8-0120-4f64-9477-e13612d21804.png)
Сделать зеркало из дисков
![](http://188.120.246.191:8098/uploads/89772cd0-220e-49a0-9f6a-65613fa2772a.png)
![](http://188.120.246.191:8098/uploads/5fd8e1c7-f404-42c9-9063-4f6f468b3ac5.png)
![](http://188.120.246.191:8098/uploads/32286c15-9beb-4ab6-b2d2-d8553f633ad3.png)
![](http://188.120.246.191:8098/uploads/8a2989f8-e2d4-4136-bc12-8f98e92da249.png)
На остальных страницах нажать Ok/Next
#### Установка NFS
![](http://188.120.246.191:8098/uploads/b421417c-e8f9-4854-b9a6-7b403ab58007.png)
Создать в новом диске папку "Storage"
![](http://188.120.246.191:8098/uploads/96321029-5057-4c0d-a476-0b19cac1e198.png)
Настроить общую папку в разделе "Files and Storage Services", в пункте "Shares" нажать "To create ...":
![](http://188.120.246.191:8098/uploads/49e4b26d-29f1-44a2-9e1f-3556517a5100.png)
Выбрать "SMB Shares - Quick"
![](http://188.120.246.191:8098/uploads/9ec9338f-1a4d-42ee-a56b-b5bdcb92b922.png)
![](http://188.120.246.191:8098/uploads/c702ff87-ddaa-49ea-8c9e-6cf8f31484b7.png)

И продолжить нажимать next 

В Tools выбрать computer management
![](http://188.120.246.191:8098/uploads/099a4c56-a342-4a3d-aa1a-63269bd9df75.png)

В Users добавить юзера/изменить пароль(Например it):
![](http://188.120.246.191:8098/uploads/aa6836a0-b4c8-463b-a985-50c389bb0604.png)

Зайти в storage
![](http://188.120.246.191:8098/uploads/1316686e-ccde-4551-95d7-d615d4bb9400.png)

И в настройках Security добавить этого пользователя
![](http://188.120.246.191:8098/uploads/92ecc858-0e0f-4315-9e75-174ad4c87a0d.png)
Нажать add
![](http://188.120.246.191:8098/uploads/2156aeb2-73f0-47f9-859e-ab31428fb8fb.png)
В этой области написать имя юзера и нажать check names
![](http://188.120.246.191:8098/uploads/7edb77b3-6107-4d0c-bcbe-0f443072988d.png)
После чего выдать котроль юзеру
![](http://188.120.246.191:8098/uploads/2a3b99d5-ae98-4a7b-bd60-6ab56ca815a6.png)

### !!! Потыкать в настройках папки и добавить куда только можно нового пользователя(Как минимум попробовать раздел с "Обмен" или "Общий доступ" или как-то так по смыслу)


### RTR-L/RTR-R

```bash

sudo firewall-cmd --permanent --zone=trusted --add-interface=ens33

sudo firewall-cmd --reload

```

### WEB-R/WEB-L
Установить ```cifs-utils```
```bash

sudo apt-get install cifs-utils -y

```
Создать папку ```/opt/share/```
```bash

mkdir /opt/share/

```

Через ```su``` в файл ```/root/.smbclient``` прописать:
```bash

user=it #Новый юзер из SRV
password=uemc #пароль от юзера

```

Дополнить файл ```/etc/fstab```:
```bash

sudo nano /etc/fstab

```
И дополнить его:
```bash
# Вместо ip лучше использовать SRV, но сначала пропингуйе его для проверки
//192.168.100.200/storage /opt/share cifs user,rw,_netdev,file_mode=0777,dir_mode=0777,credentials=/root/.smbclient 0 0

```
И применить настройки:
```bash

sudo mount -a

```
```

sudo reboot

```

Для проверки
```bash

cd /opt/share/

mkdir 123

```
## Сертификаты
### SRV
На русском название скорее всего будет по типу "Служба сертификатов Active Directory"
Установить:
![](http://188.120.246.191:8098/uploads/c48b5169-2ac3-4999-b604-30576735974d.png)
После чего начать настройку:
![](http://188.120.246.191:8098/uploads/ecb64635-68e5-4723-aa4f-cf9d457921cc.png)

![](http://188.120.246.191:8098/uploads/29a1ebc1-cdf5-4321-8398-e4b6b73de78f.png)


Написать "Demo.wsr" в:
![](http://188.120.246.191:8098/uploads/c3f07361-a8b2-4db1-b1db-195b6315e027.png)
Поставить период в 500 дней:
![](http://188.120.246.191:8098/uploads/c41d9efa-b3fd-4b03-b109-8369e8515780.png)
Установить IIS
![](http://188.120.246.191:8098/uploads/3c35de52-437a-4203-b50f-4da658e11d8b.png)
В IIS с помощью ПКМ по "SRV" открыть меню настроек "IIS Manager"
![](http://188.120.246.191:8098/uploads/183aed84-4240-45c1-8a4d-d4f1e6441e38.png)
Открыть найстройку "Bindings"
![](http://188.120.246.191:8098/uploads/9f12d5c3-0077-4f9a-bab6-2808f14b8125.png)
Нажать Add и сделать по картинке
![](http://188.120.246.191:8098/uploads/f8054405-bb29-4798-a9fc-982f5d75a863.png)
Установить все с картинки
![](http://188.120.246.191:8098/uploads/59187790-38be-466c-803d-624fc33b3449.png)
Зайти через браузер(Internet Explorer) на SRV по адресу https://localhost/certsrv и нажать Request a certificate
![](http://188.120.246.191:8098/uploads/a51a7749-4045-4436-b1d9-4e54bf92bdec.png)
Выбрать advanced certificate request > Create and submit a request to this CA и заполнить все по примеру:
![](http://188.120.246.191:8098/uploads/11981347-e1a8-43b4-9cf7-c181b41d5f4b.png)
Зайти в настройки AD CS
![](http://188.120.246.191:8098/uploads/4ee275ea-3390-4b2a-a98f-e4c5308a66b6.png)
Зайти в Pending Requests и подтвердить новый сертификат
![](http://188.120.246.191:8098/uploads/9274e3d4-80c6-458f-af3d-c4ea579b9552.png)
Снова перейти в браузер > View the status of a pending certificate request > Install this certificate
Нажать Win+R и ввести mmc
В новом окне выбрать
![](http://188.120.246.191:8098/uploads/dd959273-fb24-48ed-bd2b-ad59a9534203.png)
Выбрать Certificates и нажать Finish
![](http://188.120.246.191:8098/uploads/66750719-e216-4dd1-b475-39bea8e1a50e.png)
Зайти по пути
![](http://188.120.246.191:8098/uploads/92630ac5-f529-488c-b1bb-88f91d3bc6b8.png)
Выбрать сертификат и нажать Export
![](http://188.120.246.191:8098/uploads/d8964bb6-5670-4608-86ec-17a6ac941939.png)
Выбрать пункт "Yes"
![](http://188.120.246.191:8098/uploads/f5c9b6e7-9c98-4961-bc0d-6e82b0bd0148.png)
Задать пароль
![](http://188.120.246.191:8098/uploads/9e63ae4b-15da-40b1-bb05-1734b3d566ac.png)
Сохранить в R:/Storage с любым названием (Дальнейшие команды зависят от названия, название в примере - `www.pfx`)
![](http://188.120.246.191:8098/uploads/cb5ba0df-1bbe-40d1-8af6-b7602affd5c9.png)
### WEB-L WEB-R
Установить nginx:
```bash 

apt install -y nginx

```
Создать ключ и сертификат
```bash

cd /opt/share

openssl pkcs12 -nodes -nocerts -in www.pfx -out www.key

openssl pkcs12 -nodes -nokeys -in www.pfx -out www.cer

```
Скопировать ключ и сертификат:
```bash

cp /opt/share/www.key /etc/nginx/www.key

cp /opt/share/www.cer /etc/nginx/www.cer

```
Добавить их в файл:
```bash

nano /etc/nginx/snippets/snakeoil.conf

```
```bash

ssl_certificate /etc/nginx/www.cer;
ssl_certificate_key /etc/nginx/www.key;

```
Пример
![](http://188.120.246.191:8098/uploads/c9ae8e9c-b117-40a6-b458-6d77a2614fd7.png)

В файле `/etc/nginx/nginx.conf` сделать конфиг:
```bash

events {
    worker_connections 1024;
}

http {
    upstream backend {
        server 192.168.100.100:8080 fail_timeout=25;
        server 172.16.100.100:8080 fail_timeout=25;
    }

    server {
        listen 443 ssl default_server;
        include snippets/snakeoil.conf;

        server_name www.demo.wsr;

        location / {
            proxy_pass http://backend ;
        }
    }

    server {
        listen 80  default_server;
        server_name _;
        return 301 https://www.demo.wsr;
    }
}

```
Перезапустить nginx
```bash

sudo systemctl restart nginx

```

## Docker
### WEB-L WEB-R 
Установить docker:
```bash

sudo apt-get install docker.io

```

Запустить контейнер:
```bash

sudo docker run --name nginx -p 8080:80 -d nginx

```
Тут менять и добавить `<meta charset="utf-8">` под `<head>``:
```bash

sudo nano /usr/share/nginx/html/index.html
ыs
```
Перезапустить
```bash
sudo docker stop nginx

sudo docker rm nginx
```



```bash
sudo docker run --name nginx -p 8080:80 -v /usr/share/nginx/html/:/usr/share/nginx/html/ -d nginx
```
Без сертификатов `/etc/nginx/nginx.conf`:
```bash
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server 192.168.100.100:8080 fail_timeout=25;
        server 172.16.100.100:8080 fail_timeout=25;
    }

    server {
        listen 80  default_server;
        server_name www.demo.wsr;
        location / {
            proxy_pass http://backend ;
    }
}
}
```

`sudo systemctl restart nginx`

### Проверка
Запустить на CLI в браузере http://www.demo.wsr/

## SSH

### RTR-L
```bash

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/0 forward-port port=2222 protocol=tcp to-port=22 to-addr=192.168.100.100'

sudo firewall-cmd --reload

```

### RTR-R
```bash

sudo firewall-cmd --permanent --zone=external --add-rich-rule='rule family=ipv4 source address=0.0.0.0/0 forward-port port=2244 protocol=tcp to-port=22 to-addr=172.16.100.100'

sudo firewall-cmd --reload

```

### Проверка

### CLI
```bash

ssh user@7.7.7.100 -p 2222 # Подключиться к WEB-L

# exit | чтобы выйти

ssh user@8.8.8.100 -p 2244 # Подключиться к WEB-R 

```

## IPSec
### проверка
RTR-L RTR-R
```bash

sudo ipsec status > test
cat test

```
### RTR-R, RTR-L

```bash

sudo apt-get install libreswan -y

```

```bash

sudo apt-get install strongswan -y

```

`/etc/ipsec.d/mytunnel.conf`
```bash

conn mytunnel
      auto=start
      authby=secret
      type=tunnel
      ike=3des-sha1;modp2048
      keyexchange=ike
      phase2=esp
      phase2alg=aes-sha2
      pfs=no
      compress=no
      leftsubnets={172.16.1.0/30,192.168.100.0/24,224.0.0.0/24}
      rightsubnets={172.16.1.0/30,172.16.100.0/24,224.0.0.0/24}
      left=172.16.1.1
      right=172.16.1.2

```

`/etc/ipsec.d/mytunnel.secrets`
```bash

172.16.1.1 172.16.1.2: PSK "1234567890zxc" # пароль на 13+ символов

```

`sudo systemctl enable ipsec --now`


## Инфраструктура веб-приложения.
Данный блок подразумевает установку и настройку доступа к веб-приложению, выполненному в формате контейнера f
1. Образ Docker (содержащий веб-приложение) расположен на ISO-образе дополнительных материалов;
   - Выполните установку приложения AppDocker0;
2. Пакеты для установки Docker расположены на дополнительном ISO-образе;
3. Инструкция по работе с приложением расположена на дополнительном ISO-образе;

![изображение](https://user-images.githubusercontent.com/28905300/177527698-d6a28335-ce5c-40a5-9d35-3bfb20ac53a3.png)

![изображение](https://user-images.githubusercontent.com/28905300/177531497-5ecd22c0-0f1e-4ddc-b6bf-1f0361ab0491.png)

![изображение](https://user-images.githubusercontent.com/28905300/177531589-66a41f74-c307-472a-9b17-6364bba5913c.png)

###### #*Если файл имеет расширение просто tar, то распаковывать его не нужно! Добавляем образ с расширением tar так же, в локальный docker-репозиторий по инструкции ниже.*

![изображение](https://user-images.githubusercontent.com/28905300/177531728-3e92741f-1ab3-48f8-bdc7-849b29b11824.png)

###### #*Порт сервера может быть другим, 80 например. На этом моменте нужно заострить внимание, иначе веб-страница на CLI не откроется!*

![изображение](https://user-images.githubusercontent.com/28905300/177531935-e9f43bb9-0797-439a-914c-b0b0ab80565d.png)
##
4. Необходимо реализовать следующую инфраструктуру приложения.
   - Клиентом приложения является CLI (браузер Edge);
   - Хостинг приложения осуществляется на ВМ WEB-L и WEB-R;
   - Доступ к приложению осуществляется по DNS-имени www.demo.wsr;
     - Имя должно разрешаться во “внешние” адреса ВМ управления трафиком в обоих регионах;
     - При необходимости, для доступа к к приложению допускается реализовать реверс-прокси или трансляцию портов;

###### #*Инструкция* (Readme.txt) по работе с приложением в docker-контейнере написана на русском языке. Имеющееся ПО в Debian для просмотра и редактирования текстовых файлов не распознаёт кириллицу, поэтому целесообразнее перенести или отобразить этот файл в CLI, на Windows 10. Один из способов — отобразить содержимое инструкции через веб-страницу, при доступе к WEB-L с CLI.

![изображение](https://user-images.githubusercontent.com/28905300/177532367-0704a4d4-83dc-464f-96c2-7a6939626126.png)

###### #По указанному пути должна быть доступна инструкция. Особое внимание на используемый порт приложением.

![изображение](https://user-images.githubusercontent.com/28905300/177533093-7607ed14-ff59-45e4-8d50-fb06dacc9b31.png)

###### #Второй способ — скопировать инструкцию Readme.txt с WEB-L на CLI, через scp с использованием настроенного ранее порта 2222.

![изображение](https://user-images.githubusercontent.com/28905300/177533315-a38d56ba-94e3-45d1-bbc2-038215d47065.png)

![изображение](https://user-images.githubusercontent.com/28905300/177533634-71e97be4-e76b-4076-b850-472cf65eb969.png)
##
   - Доступ к приложению должен быть защищен с применением технологии TLS;
     - Необходимо обеспечить корректное доверие сертификату сайта, без применения “исключений” и подобных механизмов;
   - Незащищенное соединение должно переводится на защищенный канал автоматически;
5. Необходимо обеспечить отказоустойчивость приложения;
   - Сайт должен продолжать обслуживание (с задержкой не более 25 секунд) в следующих сценариях:
     - Отказ одной из ВМ Web
     - Отказ одной из ВМ управления трафиком. 