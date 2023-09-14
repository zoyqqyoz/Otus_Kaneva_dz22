Домашнее задание: VPN

1. Между двумя виртуалками поднять vpn в режимах:
- tun
- tap

Описать в чём разница, замерить скорость между виртуальными машинами в туннелях, сделать вывод об отличающихся показателях скорости.

2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку.

1. Для выполнения первого пункта необходимо написать Vagrantfile, который будет поднимать 2 виртуальные машины server и client.

```
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
config.vm.box = "centos/8"
config.vm.define "server" do |server|
server.vm.hostname = "server.loc"
server.vm.network "private_network", ip: "192.168.56.10"
end
config.vm.define "client" do |client|
client.vm.hostname = "client.loc"
client.vm.network "private_network", ip: "192.168.56.20"
end
end

neva@Uneva:~$ vagrant up
neva@Uneva:~$ vagrant status
Current machine states:

server                    running (virtualbox)
client                    running (virtualbox)
```

2. После запуска машин из Vagrantfile заходим на ВМ server и выполняем следующие действия на server и client машинах:

 устанавливаем epel репозиторий:

```
[root@client yum.repos.d]#  yum install -y epel-release
Installed:
  epel-release-8-11.el8.noarch

Complete!
```

устанавливаем пакет openvpn и iperf3

```
[root@client yum.repos.d]# yum install -y openvpn iperf3
Installed:
  iperf3-3.5-6.el8.x86_64                              lksctp-tools-1.0.18-3.el8.x86_64                              openvpn-2.4.12-1.el8.x86_64                              pkcs11-helper-1.22-7.el8.x86_64

Complete!
```

Отключаем SELinux (при желании можно написать правило для openvpn)

```
[root@server yum.repos.d]# setenforce 0
```

3. Настройка openvpn сервера:

создаём файл-ключ

```
[root@server ~]# openvpn --genkey --secret /etc/openvpn/static.key
```

создаем конфигурационный файл vpn-сервера

```
[root@server ~]# vi /etc/openvpn/server.conf

```

Файл server.conf должен содержать следующий конфиг.

```
dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

4. Создадим service unit для запуска openvpn

```
[Unit]
Description=OpenVPN Tunneling Application On %I
After=network.target

[Service]
Type=notify
PrivateTmp=true
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf

[Install]
WantedBy=multi-user.target
```

Запускаем openvpn сервер и добавляем в автозагрузку

```
[root@server ~]# systemctl start openvpn@server
[root@server ~]# systemctl enable openvpn@server
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn@server.service → /etc/systemd/system/openvpn@.service.
```

4. Настройка openvpn клиента.

Cоздаем конфигурационный файл клиента
Файл должен содержать следующий конфиг

```
dev tap
remote 192.168.56.10
ifconfig 10.10.10.2 255.255.255.0
topology subnet
route 192.168.56.0 255.255.255.0
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

На клиент в директорию /etc/openvpn необходимо скопировать файл-ключ static.key, который был создан на сервере.

Запускаем openvpn клиент и добавляем в автозагрузку:

```
[root@client ~]# systemctl start openvpn@server

[root@client ~]# systemctl enable openvpn@server
Created symlink from /etc/systemd/system/multi-user.target.wants/openvpn@server.service to /usr/lib/systemd/system/openvpn@.service.

[root@client ~]# systemctl status openvpn@server
● openvpn@server.service - OpenVPN Robust And Highly Flexible Tunneling Application On server
   Loaded: loaded (/usr/lib/systemd/system/openvpn@.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-09-04 13:23:03 UTC; 12s ago
 Main PID: 3351 (openvpn)
   Status: "Initialization Sequence Completed"
   CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
           └─3351 /usr/sbin/openvpn --cd /etc/openvpn/ --config server.conf

Sep 04 13:23:03 client systemd[1]: Starting OpenVPN Robust And Highly Flexible Tunneling Application On server...
Sep 04 13:23:03 client systemd[1]: Started OpenVPN Robust And Highly Flexible Tunneling Application On server.
```

5. Далее необходимо замерить скорость в туннеле.

на openvpn сервере запускаем iperf3 в режиме сервера

```
[root@server ~]#  iperf3 -s &
[root@server ~]# -----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 54184
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 54186
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  4.27 MBytes  35.9 Mbits/sec
[  5]   1.00-2.00   sec  5.63 MBytes  47.2 Mbits/sec
[  5]   2.00-3.00   sec  5.52 MBytes  46.3 Mbits/sec
[  5]   3.00-4.00   sec  6.33 MBytes  53.1 Mbits/sec
[  5]   4.00-5.00   sec  6.22 MBytes  52.2 Mbits/sec
[  5]   5.00-6.00   sec  6.13 MBytes  51.4 Mbits/sec
[  5]   6.00-7.00   sec  5.57 MBytes  46.7 Mbits/sec
[  5]   7.00-8.00   sec  6.08 MBytes  51.0 Mbits/sec
[  5]   8.00-9.00   sec  5.72 MBytes  48.0 Mbits/sec
[  5]   9.00-10.00  sec  6.64 MBytes  55.5 Mbits/sec
[  5]  10.00-11.00  sec  6.09 MBytes  51.3 Mbits/sec
[  5]  11.00-12.00  sec  6.25 MBytes  52.4 Mbits/sec
[  5]  12.00-13.00  sec  6.23 MBytes  52.2 Mbits/sec
[  5]  13.00-14.00  sec  5.41 MBytes  45.4 Mbits/sec
[  5]  14.00-15.00  sec  4.73 MBytes  39.7 Mbits/sec
[  5]  15.00-16.00  sec  5.84 MBytes  49.0 Mbits/sec
[  5]  16.00-17.00  sec  5.77 MBytes  48.4 Mbits/sec
[  5]  17.00-18.00  sec  6.21 MBytes  52.1 Mbits/sec
[  5]  18.00-19.00  sec  6.01 MBytes  50.4 Mbits/sec
[  5]  19.00-20.00  sec  5.80 MBytes  48.7 Mbits/sec
[  5]  20.00-21.00  sec  6.05 MBytes  50.7 Mbits/sec
[  5]  21.00-22.00  sec  6.11 MBytes  51.2 Mbits/sec
[  5]  22.00-23.00  sec  6.13 MBytes  51.5 Mbits/sec
[  5]  23.00-24.00  sec  6.03 MBytes  50.6 Mbits/sec
[  5]  24.00-25.00  sec  5.82 MBytes  48.9 Mbits/sec
[  5]  25.00-26.00  sec  5.55 MBytes  46.5 Mbits/sec
[  5]  26.00-27.00  sec  5.93 MBytes  49.7 Mbits/sec
[  5]  27.00-28.00  sec  6.00 MBytes  50.3 Mbits/sec
[  5]  28.00-29.00  sec  4.92 MBytes  41.3 Mbits/sec
[  5]  29.00-30.00  sec  4.60 MBytes  38.6 Mbits/sec
[  5]  30.00-31.00  sec  4.70 MBytes  39.4 Mbits/sec
[  5]  31.00-32.00  sec  5.08 MBytes  42.6 Mbits/sec
[  5]  32.00-33.00  sec  6.32 MBytes  53.0 Mbits/sec
[  5]  33.00-34.00  sec  6.38 MBytes  53.5 Mbits/sec
[  5]  34.00-35.00  sec  5.80 MBytes  48.6 Mbits/sec
[  5]  35.00-36.00  sec  5.56 MBytes  46.8 Mbits/sec
[  5]  36.00-37.00  sec  5.79 MBytes  48.6 Mbits/sec
[  5]  37.00-38.00  sec  5.90 MBytes  49.3 Mbits/sec
[  5]  38.00-39.00  sec  5.81 MBytes  48.8 Mbits/sec
[  5]  39.00-40.00  sec  6.47 MBytes  54.3 Mbits/sec
[  5]  40.00-40.11  sec   694 KBytes  50.9 Mbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-40.11  sec   232 MBytes  48.5 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

На openvpn клиенте запускаем iperf3 в режиме клиента и замеряем скорость в туннеле
```
[root@client yum.repos.d]# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 54186 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.02   sec  28.9 MBytes  48.3 Mbits/sec   12   95.5 KBytes
[  5]   5.02-10.01  sec  30.2 MBytes  50.7 Mbits/sec    8   92.9 KBytes
[  5]  10.01-15.02  sec  28.6 MBytes  47.8 Mbits/sec    8   80.0 KBytes
[  5]  15.02-20.01  sec  29.7 MBytes  49.9 Mbits/sec    8    108 KBytes
[  5]  20.01-25.00  sec  30.2 MBytes  50.7 Mbits/sec   15    103 KBytes
[  5]  25.00-30.01  sec  26.7 MBytes  44.8 Mbits/sec   10   91.6 KBytes
[  5]  30.01-35.02  sec  28.5 MBytes  47.7 Mbits/sec    8    102 KBytes
[  5]  35.02-40.00  sec  29.5 MBytes  49.6 Mbits/sec   11   96.8 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   232 MBytes  48.7 Mbits/sec   80             sender
[  5]   0.00-40.11  sec   232 MBytes  48.5 Mbits/sec                  receiver
```

6. Повторяем пункты 1-5 для режима работы tun. Конфигурационные файлы сервера и клиента изменятся только в директиве dev. Делаем выводы о режимах, их достоинствах и недостатках.
Вот результаты:

```
[root@server ~]# iperf3 -s &
[1] 944
[root@server ~]# -----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 54278
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 54280
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  4.22 MBytes  35.4 Mbits/sec
[  5]   1.00-2.00   sec  5.92 MBytes  49.7 Mbits/sec
[  5]   2.00-3.00   sec  6.41 MBytes  53.8 Mbits/sec
[  5]   3.00-4.00   sec  5.72 MBytes  48.0 Mbits/sec
[  5]   4.00-5.00   sec  6.39 MBytes  53.4 Mbits/sec
[  5]   5.00-6.00   sec  6.19 MBytes  52.2 Mbits/sec
[  5]   6.00-7.00   sec  5.73 MBytes  48.0 Mbits/sec
[  5]   7.00-8.00   sec  5.46 MBytes  45.8 Mbits/sec
[  5]   8.00-9.00   sec  5.19 MBytes  43.5 Mbits/sec
[  5]   9.00-10.00  sec  5.57 MBytes  46.7 Mbits/sec
[  5]  10.00-11.00  sec  5.57 MBytes  46.7 Mbits/sec
[  5]  11.00-12.00  sec  5.64 MBytes  47.3 Mbits/sec
[  5]  12.00-13.00  sec  6.10 MBytes  51.1 Mbits/sec
[  5]  13.00-14.00  sec  6.47 MBytes  54.3 Mbits/sec
[  5]  14.00-15.00  sec  6.38 MBytes  53.6 Mbits/sec
[  5]  15.00-16.01  sec  6.41 MBytes  53.4 Mbits/sec
[  5]  16.01-17.00  sec  5.79 MBytes  48.9 Mbits/sec
[  5]  17.00-18.00  sec  5.84 MBytes  49.0 Mbits/sec
[  5]  18.00-19.00  sec  5.43 MBytes  45.6 Mbits/sec
[  5]  19.00-20.01  sec  5.46 MBytes  45.3 Mbits/sec
[  5]  20.01-21.00  sec  5.26 MBytes  44.6 Mbits/sec
[  5]  21.00-22.00  sec  5.88 MBytes  49.4 Mbits/sec
[  5]  22.00-23.00  sec  6.39 MBytes  53.4 Mbits/sec
[  5]  23.00-24.00  sec  6.43 MBytes  54.0 Mbits/sec
[  5]  24.00-25.00  sec  6.29 MBytes  52.9 Mbits/sec
[  5]  25.00-26.00  sec  6.37 MBytes  53.4 Mbits/sec
[  5]  26.00-27.00  sec  6.34 MBytes  53.1 Mbits/sec
[  5]  27.00-28.00  sec  6.44 MBytes  54.1 Mbits/sec
[  5]  28.00-29.00  sec  6.75 MBytes  56.6 Mbits/sec
[  5]  29.00-30.00  sec  5.71 MBytes  47.9 Mbits/sec
[  5]  30.00-31.00  sec  6.23 MBytes  52.4 Mbits/sec
[  5]  31.00-32.00  sec  6.11 MBytes  51.2 Mbits/sec
[  5]  32.00-33.00  sec  6.72 MBytes  56.4 Mbits/sec
[  5]  33.00-34.00  sec  6.25 MBytes  52.4 Mbits/sec
[  5]  34.00-35.00  sec  6.46 MBytes  54.0 Mbits/sec
[  5]  35.00-36.00  sec  6.11 MBytes  51.4 Mbits/sec
[  5]  36.00-37.00  sec  6.55 MBytes  55.0 Mbits/sec
[  5]  37.00-38.00  sec  6.11 MBytes  51.3 Mbits/sec
[  5]  38.00-39.00  sec  6.40 MBytes  53.7 Mbits/sec
[  5]  39.00-40.00  sec  6.39 MBytes  53.6 Mbits/sec
[  5]  40.00-40.15  sec   961 KBytes  51.3 Mbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-40.15  sec   242 MBytes  50.6 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------


[root@client yum.repos.d]# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 54280 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.01   sec  29.7 MBytes  49.8 Mbits/sec   17   97.8 KBytes
[  5]   5.01-10.02  sec  28.0 MBytes  46.9 Mbits/sec    7    111 KBytes
[  5]  10.02-15.01  sec  30.4 MBytes  51.1 Mbits/sec   10    104 KBytes
[  5]  15.01-20.02  sec  28.8 MBytes  48.4 Mbits/sec   12   83.2 KBytes
[  5]  20.02-25.02  sec  30.4 MBytes  51.0 Mbits/sec    9    116 KBytes
[  5]  25.02-30.01  sec  31.3 MBytes  52.6 Mbits/sec   13   84.6 KBytes
[  5]  30.01-35.01  sec  31.9 MBytes  53.5 Mbits/sec    8   92.5 KBytes
[  5]  35.01-40.01  sec  31.7 MBytes  53.2 Mbits/sec    8   96.5 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.01  sec   242 MBytes  50.8 Mbits/sec   84             sender
[  5]   0.00-40.15  sec   242 MBytes  50.6 Mbits/sec                  receiver
```

Показатели скорости TAP и TUN оказались примерно одинаковыми, существенной разницы не наблюдается.
TAP эмулирует Ethernet устройство и работает на канальном уровне модели OSI, оперируя кадрами Ethernet, т.е. это 2-й уровень.
TUN (сетевой туннель) работает на сетевом уровне модели OSI, оперируя IP пакетами, 3-й уровень.

tap старается больше походить на реальный сетевой интерфейс, а именно он позволяет себе принимать и отправлять ARP запросы, обладает MAC адресом и может являться одним из интерфейсов сетевого моста, так как он обладает полной поддержкой ethernet - протокола канального уровня (уровень 2).
Интерфейс tun этой поддержки лишен, поэтому он может принимать и отправлять только IP пакеты и никак не ethernet кадры. Он не обладает MAC-адресом и не может быть добавлен в бридж. Зато он более легкий и быстрый за счет отсутствия дополнительной инкапсуляции и прекрасно подходит для тестирования сетевого стека или построения виртуальных частных сетей (VPN).

2. RAS на базе OpenVPN

Для выполнения данного задания можно воспользоваться Vagrantfile из 1 задания, только убрать 1 ВМ. После запуска отключаем SELinux (setenforce 0) или создаём правило для него

1. Устанавливаем репозиторий EPEL.

```
[root@server ~]# yum install -y epel-release

Upgraded:
  epel-release-8-19.el8.noarch

Complete!
```

2. Устанавливаем необходимые пакеты.

```
[root@server ~]# yum install -y openvpn easy-rsa
Installed:
  easy-rsa-3.0.8-1.el8.noarch

Complete!
```

3. Переходим в директорию /etc/openvpn/ и инициализируем pki

```
[root@server ~]# cd /etc/openvpn/
[root@server openvpn]# /usr/share/easy-rsa/3.0.8/easyrsa init-pki
init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/pki
```

4. Сгенерируем необходимые ключи и сертификаты для сервера
Предварительно проверим версию RSA:

```
[root@server openvpn]# rpm -qa | grep easy-rsa
easy-rsa-3.0.8-1.el8.noarch
[root@server openvpn]# echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass
Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020
Generating RSA private key, 2048 bit long modulus (2 primes)
.........................+++++
.................................+++++
e is 65537 (0x010001)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/pki/ca.crt

[root@server openvpn]# echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass
Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020
Generating a RSA private key
..................................................+++++
.................+++++
writing new private key to '/etc/openvpn/pki/easy-rsa-1124.cxA6Fp/tmp.3Y6d8g'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [server]:
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/pki/reqs/server.req
key: /etc/openvpn/pki/private/server.key

[root@server openvpn]# echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server
Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 825 days:

subject=
    commonName                = rasvpn


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: Using configuration from /etc/openvpn/pki/easy-rsa-1152.mk4eDd/tmp.8azKXa
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'rasvpn'
Certificate is to be certified until Dec 17 09:19:47 2025 GMT (825 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/pki/issued/server.crt

[root@server openvpn]# /usr/share/easy-rsa/3.0.8/easyrsa gen-dh
Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time

DH parameters of size 2048 created at /etc/openvpn/pki/dh.pem

[root@server openvpn]# openvpn --genkey --secret ca.key
```

5. Сгенерируем сертификаты для клиента.

```
[root@server openvpn]# echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass
Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020
Generating a RSA private key
.....................................................................+++++
......................................................................+++++
writing new private key to '/etc/openvpn/pki/easy-rsa-1251.h5YcY1/tmp.Stc3Hp'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [client]:
Keypair and certificate request completed. Your files are:
req: /etc/openvpn/pki/reqs/client.req
key: /etc/openvpn/pki/private/client.key

[root@server openvpn]# echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client
Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 825 days:

subject=
    commonName                = client


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: Using configuration from /etc/openvpn/pki/easy-rsa-1279.BDsWgL/tmp.xP75MJ
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'client'
Certificate is to be certified until Dec 17 09:24:44 2025 GMT (825 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /etc/openvpn/pki/issued/client.crt
```

6. Создадим конфигурационный файл /etc/openvpn/server.conf

```
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.10.0 255.255.255.0
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
7. Зададим параметр iroute для клиента

```
[root@server openvpn]# echo 'iroute 10.10.10.0 255.255.255.0' > /etc/openvpn/client/client
```

8. Создаём юнит, запускаем его и добавляем в автозагрузку

```
[root@server ~]# vi /etc/systemd/system/openvpn@.service
[Unit]
Description=OpenVPN Tunneling Application On %I
After=network.target

[Service]
Type=notify
PrivateTmp=true
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf

[Install]
WantedBy=multi-user.target

[root@server ~]# systemctl start openvpn@server
[root@server ~]# systemctl enable openvpn@server
```

9. Скопируем следующие файлы сертификатов и ключ для клиента на хост-
машину.
/etc/openvpn/pki/ca.crt
/etc/openvpn/pki/issued/client.crt
/etc/openvpn/pki/private/client.key
(файлы рекомендуется расположить в той же директории, что и client.conf)

10. Создадим конфигурационный файл клиента client.conf на хост-машине

```
neva@Uneva:/etc/openvpn/server$ sudo nano client.conf
dev tun
proto udp
remote 192.168.56.10 1207
client
resolv-retry infinite
remote-cert-tls server
ca ./ca.crt
cert ./client.crt
key ./client.key
route 192.168.56.0 255.255.255.0
persist-key
persist-tun
comp-lzo
verb 3
```

11. После того, как все готово, подключаемся к openvpn сервер с хост-машины.

```
neva@Uneva:/etc/openvpn/client$ sudo openvpn --config client.conf
[sudo] password for neva:
2023-09-14 13:58:02 WARNING: Compression for receiving enabled. Compression has been used in the past to break encryption. Sent packets are not compressed unless "allow-compression yes" is also set.
2023-09-14 13:58:02 --cipher is not set. Previous OpenVPN version defaulted to BF-CBC as fallback when cipher negotiation failed in this case. If you need this fallback please add '--data-ciphers-fallback BF-CBC' to your configuration and/or add BF-CBC to --data-ciphers.
2023-09-14 13:58:02 WARNING: file './client.key' is group or others accessible
2023-09-14 13:58:02 OpenVPN 2.5.5 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Jul 14 2022
2023-09-14 13:58:02 library versions: OpenSSL 3.0.2 15 Mar 2022, LZO 2.10
2023-09-14 13:58:02 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.56.10:1207
2023-09-14 13:58:02 Socket Buffers: R=[212992->212992] S=[212992->212992]
2023-09-14 13:58:02 UDP link local (bound): [AF_INET][undef]:1194
2023-09-14 13:58:02 UDP link remote: [AF_INET]192.168.56.10:1207
2023-09-14 13:58:02 TLS: Initial packet from [AF_INET]192.168.56.10:1207, sid=714b3a20 5d01da1a
2023-09-14 13:58:02 VERIFY OK: depth=1, CN=rasvpn
2023-09-14 13:58:02 VERIFY KU OK
2023-09-14 13:58:02 Validating certificate extended key usage
2023-09-14 13:58:02 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
2023-09-14 13:58:02 VERIFY EKU OK
2023-09-14 13:58:02 VERIFY OK: depth=0, CN=rasvpn
2023-09-14 13:58:02 Control Channel: TLSv1.3, cipher TLSv1.3 TLS_AES_256_GCM_SHA384, peer certificate: 2048 bit RSA, signature: RSA-SHA256
2023-09-14 13:58:02 [rasvpn] Peer Connection Initiated with [AF_INET]192.168.56.10:1207
2023-09-14 13:58:03 SENT CONTROL [rasvpn]: 'PUSH_REQUEST' (status=1)
2023-09-14 13:58:03 PUSH: Received control message: 'PUSH_REPLY,topology net30,ping 10,ping-restart 120,ifconfig 10.10.10.6 10.10.10.5,peer-id 0,cipher AES-256-GCM'
2023-09-14 13:58:03 OPTIONS IMPORT: timers and/or timeouts modified
2023-09-14 13:58:03 OPTIONS IMPORT: --ifconfig/up options modified
2023-09-14 13:58:03 OPTIONS IMPORT: peer-id set
2023-09-14 13:58:03 OPTIONS IMPORT: adjusting link_mtu to 1625
2023-09-14 13:58:03 OPTIONS IMPORT: data channel crypto options modified
2023-09-14 13:58:03 Data Channel: using negotiated cipher 'AES-256-GCM'
2023-09-14 13:58:03 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
2023-09-14 13:58:03 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
2023-09-14 13:58:03 net_route_v4_best_gw query: dst 0.0.0.0
2023-09-14 13:58:03 net_route_v4_best_gw result: via 10.30.30.1 dev ens160
2023-09-14 13:58:03 ROUTE_GATEWAY 10.30.30.1/255.255.255.0 IFACE=ens160 HWADDR=00:0c:29:41:a6:33
2023-09-14 13:58:03 TUN/TAP device tun0 opened
2023-09-14 13:58:03 net_iface_mtu_set: mtu 1500 for tun0
2023-09-14 13:58:03 net_iface_up: set tun0 up
2023-09-14 13:58:03 net_addr_ptp_v4_add: 10.10.10.6 peer 10.10.10.5 dev tun0
2023-09-14 13:58:03 net_route_v4_add: 192.168.56.0/24 via 10.10.10.5 dev [NULL] table 0 metric -1
2023-09-14 13:58:03 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
2023-09-14 13:58:03 Initialization Sequence Completed
```

12. При успешном подключении проверяем пинг по внутреннему IP адресу сервера в туннеле.

```
neva@Uneva:ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.055 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.042 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.035 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=0.041 ms
```


