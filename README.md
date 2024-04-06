## Vagrant-стенд c OSPF

### Введение

**OSPF — протокол динамической маршрутизации, использующий концепцию разделения на области в целях масштабирования.**

Административная дистанция OSPF — 110

Основные свойства протокола OSPF:

- Быстрая сходимость
- Масштабируемость (подходит для маленьких и больших сетей)
- Безопасность (поддежка аутентиикации)
- Эффективность (испольование алгоритма поиска кратчайшего пути)

При настроенном OSPF маршрутизатор формирует таблицу топологии с использованием результатов вычислений, основанных на алгоритме кратчайшего пути (SPF) Дейкстры. Алгоритм поиска кратчайшего пути основывается на данных о совокупной стоимости доступа к точке назначения. Стоимость доступа определятся на основе скорости интерфейса.

Чтобы повысить эффективность и масштабируемость OSPF, протокол поддерживает иерархическую маршрутизацию с помощью областей (area).

Область OSPF (area) — Часть сети, которой ограничивается формирование базы данных о состоянии каналов. Маршрутизаторы, находящиеся в одной и той же области, имеют одну и ту же базу данных о топологии сети. Для определения областей применяются идентификаторы областей.

Протоколы OSPF бвывают 2-х версий:

- OSPFv2
- OSPFv3

Основным отличием протоколов является то, что OSPFv2 работает с IPv4, а OSPFv3 — c IPv6.

Маршрутизаторы в OSPF классифицируются на основе выполняемой ими функции:

<img src="https://github.com/ellopa/otus-OSPF/blob/main/1.png" width=50% height=50%>

- Internal router (внутренний маршрутизатор) — маршрутизатор, все интерфейсы которого находятся в одной и той же области.
- Backbone router (магистральный маршрутизатор) — это маршрутизатор, который находится в магистральной зоне (area 0).
- ABR (пограничный маргрутизатор области) — маршрутизатор, интерфейсы которого подключены к разным областям.
- ASBR (граничный маршрутизатор автономной системы) — это маршрутизатор, у которого интерфейс подключен к внешней сети.

Также с помощью OSPF можно настроить ассиметричный роутинг.

Ассиметричная маршрутизация — возможность пересекать сеть в одном направлении, используя один путь, и возвращаться через другой путь.

## Цели домашнего задания

Создать домашнюю сетевую лабораторию. Научится настраивать протокол OSPF в Linux-based системах.

## Описание домашнего задания

1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
  - настроить OSPF между машинами на базе Quagga;
  - изобразить ассиметричный роутинг;
  - сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

Формат сдачи: Vagrantfile + ansible

Функциоанльные и нефункциональные требования

- ПК на Unix c 8 ГБ ОЗУ или виртуальная машина с включенной Nested Virtualization.

Предварительно установленное и настроенное следующее ПО:

- Hashicorp Vagrant (https://www.vagrantup.com/downloads)
- Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).
- Ansible (версия 2.8 и выше) - <https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html>
- Любой редактор кода, например Visual Studio Code, Atom и т.д.


### 1. Разворачиваем 3 виртуальные машины

Так как мы планируем настроить OSPF, все 3 виртуальные машины должны быть соединены между собой (разными VLAN), а также иметь одну (или несколько) доолнительных сетей, к которым, далее OSPF сформирует маршруты. Исходя из данных требований, мы можем нарисовать топологию сети:

<img src="https://github.com/ellopa/otus-OSPF/blob/main/2.png" width=50% height=50%>

Обратите внимание, сети, указанные на схеме не должны использоваться в Oracle Virtualbox, иначе Vagrant не сможет собрать стенд и зависнет. По умолчанию Virtualbox использует сеть 10.0.2.0/24. Если была настроена другая сеть, то проверить её можно в настройках программы: **VirtualBox — File — Preferences — Network — щёлкаем по созданной сети**

<img src="https://github.com/ellopa/otus-OSPF/blob/main/3.png" width=50% height=50%>

- Создаём каталог, в котором будут храниться настройки виртуальной машины. В каталоге создаём файл с именем [Vagrantfile](/Vagrantfile)
> В данный Vagrantfile уже добавлен модуль запуска Ansible-playbook.

- После создания данного файла, из терминала идём в каталог, в котором лежит данный Vagrantfile и вводим команду vagrant up

Результатом выполнения данной команды будут 3 созданные виртуальные машины, которые соединены между собой сетями (10.0.10.0/30, 10.0.11.0/30 и 10.0.12.0/30). У каждого роутера есть дополнительная сеть:

- на router1 — 192.168.10.0/24
- на router2 — 192.168.20.0/24
- на router3 — 192.168.30.0/24

На данном этапе ping до дополнительных сетей (192.168.10-30.0/24) с соседних роутеров будет недоступен.
Для подключения к ВМ нужно ввести команду vagrant ssh <имя машины>, например vagrant ssh router1
Далее потребуется переключиться в root пользователя: sudo -i
Далее все примеры команд будут указаны от пользователя root.

### Первоначальная настройка Ansible

Для настроки хостов с помощью Ansible нам нужно создать несколько файлов и  положить их в отдельную папку (например ansible):

- Конфигурационный файл: ansible.cfg — файл описывает базовые настройки для работы Ansible:
```bash
[defaults]
# Отключение проверки ключа хоста
host_key_checking = false
# Указываем имя файла инвентаризации
inventory = hosts
# Отключаем игнорирование предупреждений
command_warnings= false
```
- Файл инвентаризации hosts — данный файл хранит информацию о том, как подключиться к хосту:
```bash
[routers]
router1 ansible_host=192.168.56.10 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router1/virtualbox/private_key
router2 ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router2/virtualbox/private_key
router3 ansible_host=192.168.56.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router3/virtualbox/private_key
```

>- [servers] - в квадратных скобках указана группа хостов
>- router1 — имя нашего хоста (имена хостов и групп не могут быть одинаковые)
>- ansible_host — адрес нашего хоста
>- ansible_user — имя пользователя, с помощью которого Ansible будет подключаться к хосту
>- ansible_ssh_private_key — адрес расположения ssh-ключа
>- В файл инвентаризации также можно добовлять переменные, которые могут автоматически добавляться в jinja template. Добавление переменных будет рассмотрено далее.

- Ansible-playbook provision.yml — основной файл, в котором содержатся инструкции (модули) по настройке для Ansible
- Дополнительно можно создать каталоги для темплейтов конфигурационных файлов (templates) и файлов с переменными (defaults)

### Установка пакетов для тестирования и настройки OSPF

- Перед настройкой FRR рекомендуется поставить базовые программы для изменения конфигурационных файлов (vim) и изучения сети (traceroute, tcpdump, net-tools):

```bash
apt update
apt install vim traceroute tcpdump net-tools
```
- Пример установки пакетов с помощью Ansible

```yml
---
# Начало файла provision.yml
- name: OSPF
  # У казываем имя хоста или группу, которые будем настраивать
  hosts: all
  # Параметр выполнения модулей от root-пользователя
  become: true
  # Указание файла с дополнителыми переменными (понадобится при добавлении темплейтов)
  vars_files:
    - defaults/main.yml
  tasks:
    # Обновление пакетов и установка vim, traceroute, tcpdump, net-tools
    - name: install base tools
      apt:
        name:
          - vim
          - traceroute
          - tcpdump
          - net-tools
        state: present
        update_cache: true
```

### 2.1 Настройка OSPF между машинами на базе Quagga

**Пакет Quagga перестал развиваться в 2018 году. Ему на смену пришёл пакет FRR, он построен на базе Quagga и продолжает своё развитие. В данном руководстве настойка OSPF будет осуществляться в FRR.**

- Процесс установки FRR и настройки OSPF вручную:
  - Отключить файерволл ufw и удалить его из автозагрузки:
```bash
systemctl stop ufw
systemctl disable ufw
```
```
vagrant@router1:~$ sudo -i
root@router1:~# systemctl stop ufw
root@router1:~# systemctl disable ufw
Synchronizing state of ufw.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable ufw
Removed /etc/systemd/system/multi-user.target.wants/ufw.service.
```
- Добавить gpg ключ:
```bash
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
```
```
root@router1:~# curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -

Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
```
>- GPG (также известный как GnuPG) создавался как свободная альтернатива несвободному PGP. GPG используется для шифрования информации и предоставляет различные алгоритмы (RSA, DSA, AES и др.) для решения этой задачи.
>- GPG может использоваться для симметричного шифрования, но в основном программа используется для ассиметричного шифрования информации. Если кратко — при симметричном шифровании для шифровки и расшифровки сообщения используется один ключ (например, какой символ соответствует той или иной букве). При ассиметричном шифровании используются 2 ключа — публичный и приватный. Публичный используется для шифрования и его мы можете дать своим друзьям, а приватны — для расшифровки, и его вы должны хранить в безопасности. Благодаря такой схеме расшифровать сообщение может только владелец приватного ключа (даже тот,кто зашифровывал сообщение, не может произвести обратную операцию).
 
- Добавить репозиторий c пакетом FRR:
```bash
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list
```
```
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list
```
- Обновить пакеты и устанавить FRR:
```bash
sudo apt update
sudo apt install frr frr-pythontools -y
```
```
root@router1:~# apt install frr frr-pythontools -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libc-ares2 libprotobuf-c1 libyang2
Suggested packages:
  frr-doc
........
Processing triggers for libc-bin (2.35-0ubuntu3.4) ...
Scanning processes...                                                                                                       
Scanning linux images...                                                                                                    

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.

- Разрешаем (включаем) маршрутизацию транзитных пакетов:

```bash
sysctl net.ipv4.conf.all.forwarding=1
```
- Включить демон ospfd в FRR. Для этого открыть в редакторе файл /etc/frr/daemons и изменить в нём параметры для пакетов zebra и ospfd на yes: vim /etc/frr/daemons

```bash
zebra=yes
ospfd=yes
bgpd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
```
> В примере показана только часть файла

```
root@router1:~# vim /etc/frr/daemons
root@router1:~# cat /etc/frr/daemons
# This file tells the frr package which daemons to start.
#
# Sample configurations for these daemons can be found in
# /usr/share/doc/frr/examples/.
#
# ATTENTION:
#
# When activating a daemon for the first time, a config file, even if it is
# empty, has to be present *and* be owned by the user and group "frr", else
# the daemon will not be started by /etc/init.d/frr. The permissions should
# be u=rw,g=r,o=.
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= though. Check /etc/pam.d/frr, too.
#
# The watchfrr, zebra and staticd daemons are always started.
#
zebra=yes
bgpd=no
ospfd=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
pim6d=no
ldpd=no
nhrpd=no
```
> Показана только часть файла

- Настройка OSPF. Для настройки OSPF нам потребуется создать файл /etc/frr/frr.conf который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Пример создания файла на хосте router1.

- Необходимо узнать имена интерфейсов и их адреса. Сделать это можно с помощью двух способов: 
   - посмотреть в linux: ip a | grep inet
```bash
root@router1:~# ip a | grep "inet "
    inet 127.0.0.1/8 scope host lo
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic eth0
    inet 10.0.10.1/30 brd 10.0.10.3 scope global eth1
    inet 10.0.12.1/30 brd 10.0.12.3 scope global eth2
    inet 192.168.10.1/24 brd 192.168.10.255 scope global eth3
    inet 192.168.56.10/24 brd 192.168.56.255 scope global eth4
```
   - Зайти в FRR и посмотреть информацию об интерфейсах
```bash
root@router1:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
eth0            up      default         10.0.2.15/24
eth1            up      default         10.0.10.1/30
eth2            up      default         10.0.12.1/30
eth3            up      default         192.168.10.1/24
eth4            up      default         192.168.56.10/24
lo              up      default         

router1# exit

```
- Создать файл /etc/frr/frr.conf и внести в него следующую информацию для router1:
```bash
!Указание версии FRR
frr version 9.1
frr defaults traditional
!Указываем имя машины
hostname router1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе eth1 
interface eth1
  !Указываем имя интерфейса
  description r1-r2
  !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
  ip address 10.0.10.1/30
  !Указываем параметр игнорирования MTU
  ip ospf mtu-ignore
  !Если потребуется, можно указать «стоимость» интерфейса
  !ip ospf cost 1000
  !Указываем параметры hello-интервала для OSPF пакетов
  ip ospf hello-interval 10
  !Указываем параметры dead-интервала для OSPF пакетов
  !Должно быть кратно предыдущему значению
  ip ospf dead-interval 30
!
interface eth2
  description r1-r3
  ip address 10.0.12.1/30
  ip ospf mtu-ignore
  !ip ospf cost 45
  ip ospf hello-interval 10
  ip ospf dead-interval 30
!  
interface eth3
  description net_router1
  ip address 192.168.10.1/24
  ip ospf mtu-ignore
  !ip ospf cost 45
  ip ospf hello-interval 10
  ip ospf dead-interval 30
!
!Начало настройки OSPF
router ospf
  !Указываем router-id
  router-id 1.1.1.1
  !Указываем сети, которые хотим анонсировать соседним роутерам
  network 10.0.10.0/30 area 0
  network 10.0.12.0/30 area 0
  network 192.168.10.0/24 area 0
  !Указываем адреса соседних роутеров
  neighbor 10.0.10.2
  neighbor 10.0.12.2
!  
!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always
```
> - Вместо файла frr.conf можно задать данные параметры вручную из vtysh. Vtysh использует cisco-like команды.
> - На хостах router2 и router3 также потребуется настроить конфигруационные файлы, предварительно поменяв ip -адреса интерфейсов.

- Файл /etc/frr/frr.conf для router2
```bash
!Указание версии FRR
frr version 9.1
frr defaults traditional
!Указываем имя машины
hostname router2
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе eth1
interface eth1
  !Указываем имя интерфейса
  description r2-r1
  !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
  ip address 10.0.10.2/30
  !Указываем параметр игнорирования MTU
  ip ospf mtu-ignore
  !Если потребуется, можно указать «стоимость» интерфейса
  !ip ospf cost 1000
  !Указываем параметры hello-интервала для OSPF пакетов
  ip ospf hello-interval 10
  !Указываем параметры dead-интервала для OSPF пакетов
  !Должно быть кратно предыдущему значению
  ip ospf dead-interval 30
!
interface eth2
  description r2-r3
  ip address 10.0.11.2/30
  ip ospf mtu-ignore
  !ip ospf cost 45
  ip ospf hello-interval 10
  ip ospf dead-interval 30
!  
interface eth3
  description net_router2
  ip address 192.168.20.1/24
  ip ospf mtu-ignore
  !ip ospf cost 45
  ip ospf hello-interval 10
  ip ospf dead-interval 30
!
!Начало настройки OSPF
router ospf
  !Указываем router-id
  router-id 2.2.2.2
  !Указываем сети, которые хотим анонсировать соседним роутерам
  network 10.0.10.0/30 area 0
  network 10.0.11.0/30 area 0
  network 192.168.20.0/24 area 0
  !Указываем адреса соседних роутеров
  neighbor 10.0.10.1
  neighbor 10.0.11.1
!  
!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always
```
- Файл /etc/frr/frr.conf для router3
```bash
!Указание версии FRR
frr version 9.1
frr defaults traditional
!Указываем имя машины
hostname router3
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе eth1
interface eth1
  !Указываем имя интерфейса
  description r3-r2
  !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
  ip address 10.0.11.1/30
  !Указываем параметр игнорирования MTU
  ip ospf mtu-ignore
  !Если потребуется, можно указать «стоимость» интерфейса
  !ip ospf cost 1000
  !Указываем параметры hello-интервала для OSPF пакетов
  ip ospf hello-interval 10
  !Указываем параметры dead-интервала для OSPF пакетов
  !Должно быть кратно предыдущему значению
  ip ospf dead-interval 30
!
interface eth2
  description r3-r2
  ip address 10.0.12.2/30
  ip ospf mtu-ignore
  !ip ospf cost 45
  ip ospf hello-interval 10
  ip ospf dead-interval 30
!  
interface eth3
  description net_router3
  ip address 192.168.30.1/24
  ip ospf mtu-ignore
  !ip ospf cost 45
  ip ospf hello-interval 10
  ip ospf dead-interval 30
!
!Начало настройки OSPF
router ospf
  !Указываем router-id
  router-id 3.3.3.3
  !Указываем сети, которые хотим анонсировать соседним роутерам
  network 10.0.11.0/30 area 0
  network 10.0.12.0/30 area 0
  network 192.168.30.0/24 area 0
  !Указываем адреса соседних роутеров
  neighbor 10.0.11.2
  neighbor 10.0.12.1
!  
!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always
```
- В ходе создания файла нужно настроить OSPF-параметры:

> - hello-interval — интервал который указывает через сколько секунд протокол OSPF будет повторно отправлять запросы на другие роутеры. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF.
> - Dead-interval — если в течении заданного времени роутер не отвечает на запросы, то он считается вышедшим из строя и пакеты уходят на другой роутер (если это возможно). Значение должно быть кратно hello-интервалу. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF.
> - router-id — идентификатор маршрутизатора (необязательный параметр), если данный параметр задан, то роутеры определяют свои роли по данному параметру. Если данный идентификатор не задан, то роли маршрутизаторов определяются с помощью Loopback-интерфейса или самого большого ip-адреса на роутере.
> - После создания файлов /etc/frr/frr.conf и /etc/frr/daemons нужно проверить, что владельцем файла является пользователь frr. Группа файла также должна быть frr. 
> -Должны быть установленны следующие права: у владельца на чтение и запись, у группы только на чтение
```bash
ls -l /etc/frr
```
```
root@router1:~# ls -l /etc/frr
total 24
-rw-r----- 1 frr frr 4138 Mar 31 08:47 daemons
-rw-r----- 1 frr frr 2418 Mar 31 12:09 frr.conf
-rw-r----- 1 frr frr 6615 Nov 27 14:24 support_bundle_commands.conf
-rw-r----- 1 frr frr   32 Nov 27 14:24 vtysh.conf
```
> - Если права или владелец файла указан неправильно, то нужно поменять владельца и назначить правильные права, например:
```bash
chown frr:frr /etc/frr/frr.conf
chmod 640 /etc/frr/frr.conf
```
- Перезапускаем FRR и добавляем его в автозагрузку
```bash
systemctl restart frr
systemctl enable frr
```
```
root@router1:~# systemct restart frr 
Command 'systemct' not found, did you mean:
  command 'systemctl' from deb systemd (249.11-0ubuntu3.11)
  command 'systemctl' from deb systemctl (1.4.4181-1.1)
Try: apt install <deb name>
root@router1:~# systemctl enable frr
Synchronizing state of frr.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable frr
```
- Проверям, что OSPF перезапустился без ошибок
```bash
root@router1:~# systemctl status frr
● frr.service - FRRouting
     Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2024-03-31 08:42:58 UTC; 3h 39min ago
       Docs: https://frrouting.readthedocs.io/en/latest/setup.html
   Main PID: 3745 (watchfrr)
     Status: "FRR Operational"
      Tasks: 8 (limit: 2220)
     Memory: 12.1M
        CPU: 1.971s
     CGroup: /system.slice/frr.service
             ├─3745 /usr/lib/frr/watchfrr -d -F traditional zebra mgmtd staticd
             ├─3755 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s 90000000
             ├─3760 /usr/lib/frr/mgmtd -d -F traditional -A 127.0.0.1
             └─3762 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1

Mar 31 08:42:58 router1 staticd[3762]: [VTVCM-Y2NW3] Configuration Read in Took: 00:00:00
Mar 31 08:42:58 router1 frrinit.sh[3782]: [3782|staticd] done
Mar 31 08:42:58 router1 zebra[3755]: [VTVCM-Y2NW3] Configuration Read in Took: 00:00:00
Mar 31 08:42:58 router1 frrinit.sh[3766]: [3766|zebra] done
Mar 31 08:42:58 router1 watchfrr[3745]: [QDG3Y-BY5TN] zebra state -> up : connect succeeded
Mar 31 08:42:58 router1 watchfrr[3745]: [QDG3Y-BY5TN] mgmtd state -> up : connect succeeded
Mar 31 08:42:58 router1 watchfrr[3745]: [QDG3Y-BY5TN] staticd state -> up : connect succeeded
Mar 31 08:42:58 router1 frrinit.sh[3735]:  * Started watchfrr
Mar 31 08:42:58 router1 systemd[1]: Started FRRouting.
Mar 31 08:42:58 router1 watchfrr[3745]: [KWE5Q-QNGFC] all daemons up, doing startup-complete notify
```
- Проверить что с любого хоста доступны сети:
   - 192.168.10.0/24
   - 192.168.20.0/24
   - 192.168.30.0/24
   - 10.0.10.0/30
   - 10.0.11.0/30
   - 10.0.13.0/30
   
- С хоста router1: сделать ping до ip-адреса 192.168.30.1
```bash
root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=0.325 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=0.383 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=64 time=0.371 ms

--- 192.168.30.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2046ms
rtt min/avg/max/mdev = 0.325/0.359/0.383/0.025 ms

```
- Запустить трассировку до адреса 192.168.30.1
```bash
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.203 ms  0.174 ms  0.205 ms

```
- Отключить интерфейс eth2 и немного подождать и снова запустить трассировку до ip-адреса 192.168.30.1
```bash
root@router1:~# ifconfig eth2 down
root@router1:~# ip a | grep eth2
4: eth2: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.300 ms  0.263 ms  0.294 ms
 2  192.168.30.1 (192.168.30.1)  0.896 ms  0.830 ms  0.793 ms
```
>- После отключения интерфейса сеть 192.168.30.0/24 доступна.

- Также можно проверить из интерфейса vtysh какие маршруты видны на данный момент:
```bash
root@router1:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1000] is directly connected, eth1, weight 1, 00:03:08
O>* 10.0.11.0/30 [110/1100] via 10.0.10.2, eth1, weight 1, 00:03:08
O>* 10.0.12.0/30 [110/1200] via 10.0.10.2, eth1, weight 1, 00:03:08
O   192.168.10.0/24 [110/100] is directly connected, eth3, weight 1, 00:16:59
O>* 192.168.20.0/24 [110/1100] via 10.0.10.2, eth1, weight 1, 00:03:08
O>* 192.168.30.0/24 [110/1200] via 10.0.10.2, eth1, weight 1, 00:03:08
```
- Включить интерфейс обратно

```bash
root@router1:~# ifconfig eth2 up
root@router1:~# ip a | grep eth2
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 10.0.12.1/30 brd 10.0.12.3 scope global eth2
```
### Настройка OSPF c помощью Ansible

- Файл [Ansible](/provision.yml)
>- Файлы daemons и frr.conf должны лежать в каталоге ansible/templates.

>- Содержимое файла daemons одинаково на всех хостах, а вот содержание файла frr.conf на всех хостах будет разное.
>- Для того, чтобы не создавать 3 похожих файла, можно воспользоваться jinja2 template. Jinja2 позволит добавлять в файл уникальные значения для разных серверов.
>- Перед тем, как начать настройку хостов Ansible забирает информацию о каждом хосте. Эту информацию можно использовать в jinja2 темплейтах.
>- Посмотреть, какую информацию о сервере собрал Ansible можно с помощью команды:

```bash
ansible router1 -i hosts -m setup -e "host_key_checking=false"
```
>- Команда выполнятся из каталога проекта (где находится Vagrantfile).
>- Помимо фактов Jinja2 может брать информацию из файлов hosts и defaults/main.yml (так как мы указали его в начале файла provision.yml)

- Файл [jinja2](/templates/frr.conf.j2):

>- В файле frr.conf у нас есть строка: hostname router1  
>- При копировании файла на router2 и router3 нужно будет поменять имя - сделать это можно так: hostname {{ ansible_hostname }} 
>- ansible_hostname - заберет из фактов имя хостов и подставит нужное имя
>- Допустим мы хотим перед настройкой OSPF выбрать, будем ли мы указывать router-id. Для этого в файле defaults/main.yml создаем переменную router_id_enable: false
>- По умолчанию назначаем ей значение **false**

>- Значение router_id мы можем задать в файле hosts, дописав его в конец строки наших хостов:

```bash
[routers]
router1 ansible_host=192.168.56.10 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router1/virtualbox/private_key router_id=1.1.1.1
router2 ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router2/virtualbox/private_key router_id=2.2.2.2
router3 ansible_host=192.168.56.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router3/virtualbox/private_key router_id=3.3.3.3
```
>- В темплейте файла frr.conf нкжно указать условие:

```bash
{% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
```
>- Данное правило в любом случае будет подставлять значение router-id, но, если параметр router_id_enable будет равен значению false, то в начале строки будет ставиться символ ! (восклицательный знак) и строка не будет учитываться в настройках FRR.

>- Более подробно о создании темплейтов можно прочитать по этой ссылке - <https://docs.ansible.com/ansible/2.9/user_guide/playbooks_templating.html>

### 2.2 Настройка ассиметричного роутинга

>- Для настройки ассиметричного роутинга необходимо выключить блокировку ассиметричной маршрутизации: sysctl net.ipv4.conf.all.rp_filter=0

- Выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например поменяем стоимость интерфейса eth1 на router1:
```bash
root@router1:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

conf t
int eth1
ip ospf cost 1000
exit
exit
router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/300] via 10.0.12.2, eth2, weight 1, 00:23:32
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, eth2, weight 1, 00:23:32
O   10.0.12.0/30 [110/100] is directly connected, eth2, weight 1, 00:24:07
O   192.168.10.0/24 [110/100] is directly connected, eth3, weight 1, 00:24:07
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, eth2, weight 1, 00:23:32
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, eth2, weight 1, 00:23:32

```
- На router2
```bash
root@router2:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, eth1, weight 1, 00:27:43
O   10.0.11.0/30 [110/100] is directly connected, eth2, weight 1, 00:27:43
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, eth1, weight 1, 00:26:58
  *                        via 10.0.11.1, eth2, weight 1, 00:26:58
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, eth1, weight 1, 00:27:08
O   192.168.20.0/24 [110/100] is directly connected, eth3, weight 1, 00:27:43
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, eth2, weight 1, 00:27:08
```
- После внесения данных настроек, мы видим, что маршрут до сети 192.168.20.0/30 теперь пойдёт через router3, но обратный трафик от router2 пойдёт по другому пути.

Давайте это проверим:

- На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: ping -I 192.168.10.1 192.168.20.1
- На router2 запускаем tcpdump, который будет смотреть трафик только на порту eth2:

```bash
tcpdump -i  eth2

06:36:07.528682 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 758, length 64
06:36:08.552719 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 759, length 64
06:36:09.580763 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 760, length 64
06:36:10.603791 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 761, length 64
06:36:11.629778 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 762, length 64
06:36:12.652745 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 763, length 64
06:36:13.685593 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 764, length 64
06:36:14.697135 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 765, length 64
06:36:15.720708 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 766, length 64
```
> Видим что данный порт только получает ICMP-трафик с адреса 192.168.10.1

- На router2 запускаем tcpdump, который будет смотреть трафик только на порту eth1:
```bash
tcpdump -i eth1

tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
06:26:17.177030 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
06:26:17.177713 IP 10.0.10.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
06:26:17.705206 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 182, length 64
06:26:18.730931 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 183, length 64
06:26:19.753020 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 184, length 64
06:26:20.777939 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 185, length 64
06:26:21.802708 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 186, length 64
06:26:22.824963 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 187, length 64
06:26:23.849742 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 188, length 64
06:26:24.874679 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 189, length 64
06:26:25.897626 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 190, length 64
06:26:26.922280 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 191, length 64
```
> Видим что данный порт только отправляет ICMP-трафик на адрес 192.168.10.1
- Таким образом мы видим ассиметричный роутинг.

### Настройка ассиметричного роутинга с помощью Ansible

- Файл [Ansible](/provision_asr.yml) добавлено:

```yml
    # Отключаем запрет ассиметричного роутинга
    - name: set up asynchronous routing
      sysctl:
        name: net.ipv4.conf.all.rp_filter
        value: '0'
        state: present
    # Делаем интерфейс enp0s8 в router1 «дорогим»
    - name: set up OSPF
      template:
        src: frr.conf.j2
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: 0640
    # Применяем настройки
    - name: restart FRR
      service:
        name: frr
        state: restarted
        enabled: true
```
- Пример добавления «дорогого» интерфейса в template frr.conf [jinja2](/templates/frr.conf_asr.j2) добавлено:

```bash
{% if ansible_hostname == 'router1' %}
  ip ospf cost 1000
{% else %}
  !ip ospf cost 450
{% endif %}
```
```
ansible-playbook provision_asr.yml
```
> В данном примере, проверяется имя хоста, и, если имя хоста «router1», то в настройку интерфейса eth1
 добавляется стоимость 1000, в остальных случаях настройка комментируется…

### 2.3 Настройка симметичного роутинга

- Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация.В прошлом задании router2 настроен так, что будет отправлять обратно трафик через порт eth1, нужно сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:

Поменяем стоимость интерфейса eth1 на router2:

```bash
root@router2:~# vtysh

Hello, this is FRRouting (version 9.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# conf t
router2(config)# int eth1
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1000] is directly connected, eth1, weight 1, 00:00:29
O   10.0.11.0/30 [110/100] is directly connected, eth2, weight 1, 00:57:23
O>* 10.0.12.0/30 [110/200] via 10.0.11.1, eth2, weight 1, 00:00:29
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, eth2, weight 1, 00:00:29
O   192.168.20.0/24 [110/100] is directly connected, eth3, weight 1, 00:57:23
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, eth2, weight 1, 00:56:48

exit
```
- После внесения данных настроек, маршрут до сети 192.168.10.0/30 должен пойти через router3, проверим:

- На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: ping -I 192.168.10.1 192.168.20.1
- На router2 запускаем tcpdump, который будет смотреть трафик только на порту eth2:

```bash
root@router2:~# tcpdump -i  eth2


06:49:26.152788 IP 192.168.10.1 > router2: ICMP echo request, id 3, seq 31, length 64
06:49:26.152817 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 31, length 64
06:49:27.176742 IP 192.168.10.1 > router2: ICMP echo request, id 3, seq 32, length 64
06:49:27.176767 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 32, length 64
06:49:28.200782 IP 192.168.10.1 > router2: ICMP echo request, id 3, seq 33, length 64
06:49:28.200814 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 33, length 64
06:49:29.224757 IP 192.168.10.1 > router2: ICMP echo request, id 3, seq 34, length 64
06:49:29.224779 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 34, length 64
06:49:30.248762 IP 192.168.10.1 > router2: ICMP echo request, id 3, seq 35, length 64
06:49:30.248787 IP router2 > 192.168.10.1: ICMP echo reply, id 3, seq 35, length 64
```
> Теперь мы видим, что трафик между роутерами ходит симметрично.

### Настройка симметричного роутинга с помощью Ansible

- Настройка симметричного роутинга заключается в том, чтобы добавить правильную настройку в файл /etc/frr/frr.conf Далее, файл необходимо также отправить на хосты и перезапустить FRR.
- Чтобы было легко переключаться между ассиметричным и симметричным роутингом, можно добавить переменную symmetric_routing со значением по умолчанию false в файл [defaults/main.yml](/defaults/main.yml)
```
symmetric_routing: false
```
- Далее в [template](/templates/frr.conf_sr.j2) frr.conf добавить условие:

```bash
{% if ansible_hostname == 'router1' %}
  ip ospf cost 1000
{% elif ansible_hostname == 'router2' and symmetric_routing == true %}
  ip ospf cost 1000
{% else %}
  !ip ospf cost 45
{% endif %}
```
>- Данное условие проверят имя хоста и переменную symmetric_routing и добавляет в темплейт следующие параметры:
    - Если имя хоста router1 — то добавляется стоимость интерфейса 1000
    - Если имя хоста router2 И значение параметра symmetric_routing true — то добавляется стоимость интерфейса 1000
    - В остальных случаях добавляется закомментированный параметр

- Для удобного переключения параметров нам потребуется запускать из ansible-playbook только 2 последних модуля. Чтобы не ждать выполнения всего плейбука, можно добавить тег "setup_ospf" к модулям.
```bash
    - name: set up OSPF
      template:
        src: frr.conf.j2
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: 0640
      tags:
        - setup_ospf
    - name: restart FRR
      service:
        name: frr
        state: restarted
        enabled: true
      tags:
        - setup_ospf
```
- Тогда можно будет запускать provision_sr.yml не полностью. Пример запуска модулей из ansible-playbook, которые помечены тегами:

```bash
ansible-playbook -i ansible/hosts -l all provision_sr.yml -t setup_ospf -e "host_key_checking=false"
```
-  Получится [Playbook](/provision_sr.yml)
 
## Рекомендуемые источники

- [Статья «OSPF»](https://ru.bmstu.wiki/OSPF_(Open_Shortest_Path_First)#.D0.A2.D0.B5.D1.80.D0.BC.D0.B8.D0.BD.D1.8B_.D0.B8_.D0.BE.D1.81.D0.BD.D0.BE.D0.B2.D0.BD.D1.8B.D0.B5_.D0.BF.D0.BE.D0.BD.D1.8F.D1.82.D0.B8.D1.8F_OSPF)
- [Статья «IP Routing: OSPF Configuration Guide»](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/xe-16/iro-xe-16-book/iro-cfg.html)
- [Документация FRR](http://docs.frrouting.org/en/latest/overview.html)
- [Статья «Принципы таблицы маршрутизации. Асимметричная маршрутизация»](http://marshrutizatciia.ru/principy-tablicy-marshrutizacii-asimmetrichnaya-marshrutizaciya.html)
- [Различия межлу протоколами OSPF](https://da2001.ru/CCNA_5.02/2/course/module8/8.3.1.3/8.3.1.3.html#:~:text=%D0%9A%D0%BE%D0%BD%D1%84%D0%B8%D0%B3%D1%83%D1%80%D0%B0%D1%86%D0%B8%D1%8F%20OSPFv3%20%D0%B4%D0%BB%D1%8F%20%D0%BE%D0%B4%D0%BD%D0%BE%D0%B9%20%D0%BE%D0%B1%D0%BB%D0%B0%D1%81%D1%82%D0%B8&text=%D0%98%D1%81%D1%85%D0%BE%D0%B4%D0%BD%D1%8B%D0%B9%20%D0%B0%D0%B4%D1%80%D0%B5%D1%81%20%E2%80%94%20%D1%81%D0%BE%D0%BE%D0%B1%D1%89%D0%B5%D0%BD%D0%B8%D1%8F%20OSPFv2%20%D0%BF%D0%BE%D1%81%D1%82%D1%83%D0%BF%D0%B0%D1%8E%D1%82,%D1%82%D0%B8%D0%BF%D0%B0%20link%2Dlocal%20%D0%B2%D1%8B%D1%85%D0%BE%D0%B4%D0%BD%D0%BE%D0%B3%D0%BE%20%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D1%84%D0%B5%D0%B9%D1%81%D0%B0)
- [Документация по Templating(jinja2)](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_templating.html)
