Стенд с Vagrant c Rsyslog

Цель домашнего задания
Научится проектировать централизованный сбор логов. Рассмотреть особенности разных платформ для сбора логов.

Описание домашнего задания
1. В Vagrant разворачиваем 2 виртуальные машины web и log
2. на web настраиваем nginx
3. на log настраиваем центральный лог сервер на любой системе на выбор
journald;
rsyslog;
elk.
4. настраиваем аудит, следящий за изменением конфигов nginx 

Все критичные логи с web должны собираться и локально и удаленно.
Все логи с nginx должны уходить на удаленный сервер (локально только критичные).
Логи аудита должны также уходить на удаленную систему.

1. Создаём 2 ВМ web и log на основе приложенного [Vagrantfile](https://github.com/zoyqqyoz/Otus_Kaneva_dz16/blob/master/Vagrantfile) и запускаем их:

```
neva@Uneva:~$ vagrant up
Bringing machine 'web' up with 'virtualbox' provider...
Bringing machine 'log' up with 'virtualbox' provider...
==> web: Box 'centos/7' could not be found. Attempting to find and install...
    web: Box Provider: virtualbox
    web: Box Version: 2004.01
==> web: Loading metadata for box 'centos/7'
    web: URL: https://vagrantcloud.com/centos/7
==> web: Adding box 'centos/7' (v2004.01) for provider: virtualbox
    web: Downloading: https://vagrantcloud.com/centos/boxes/7/versions/2004.01/providers/virtualbox.box
Download redirected to host: cloud.centos.org
    web: Calculating and comparing box checksum...
==> web: Successfully added box 'centos/7' (v2004.01) for 'virtualbox'!
==> web: Importing base box 'centos/7'...
==> web: Matching MAC address for NAT networking...
==> web: Checking if box 'centos/7' version '2004.01' is up to date...
==> web: Setting the name of the VM: neva_web_1690875699907_48376
==> web: Fixed port collision for 22 => 2222. Now on port 2200.
==> web: Clearing any previously set network interfaces...
==> web: Preparing network interfaces based on configuration...
    web: Adapter 1: nat
    web: Adapter 2: hostonly
==> web: Forwarding ports...
    web: 22 (guest) => 2200 (host) (adapter 1)
==> web: Running 'pre-boot' VM customizations...
==> web: Booting VM...
==> web: Waiting for machine to boot. This may take a few minutes...
    web: SSH address: 127.0.0.1:2200
    web: SSH username: vagrant
    web: SSH auth method: private key
    web:
    web: Vagrant insecure key detected. Vagrant will automatically replace
    web: this with a newly generated keypair for better security.
    web:
    web: Inserting generated public key within guest...
    web: Removing insecure key from the guest if it's present...
    web: Key inserted! Disconnecting and reconnecting using new SSH key...
==> web: Machine booted and ready!
==> web: Checking for guest additions in VM...
    web: No guest additions were detected on the base box for this VM! Guest
    web: additions are required for forwarded ports, shared folders, host only
    web: networking, and more. If SSH fails on this machine, please install
    web: the guest additions and repackage the box to continue.
    web:
    web: This is not an error message; everything may continue to work properly,
    web: in which case you may ignore this message.
==> web: Setting hostname...
==> web: Configuring and enabling network interfaces...
==> log: Box 'centos/7' could not be found. Attempting to find and install...
    log: Box Provider: virtualbox
    log: Box Version: 2004.01
==> log: Loading metadata for box 'centos/7'
    log: URL: https://vagrantcloud.com/centos/7
==> log: Adding box 'centos/7' (v2004.01) for provider: virtualbox
==> log: Importing base box 'centos/7'...
==> log: Matching MAC address for NAT networking...
==> log: Checking if box 'centos/7' version '2004.01' is up to date...
==> log: Setting the name of the VM: neva_log_1690875747658_20530
==> log: Fixed port collision for 22 => 2222. Now on port 2201.
==> log: Clearing any previously set network interfaces...
==> log: Preparing network interfaces based on configuration...
    log: Adapter 1: nat
    log: Adapter 2: hostonly
==> log: Forwarding ports...
    log: 22 (guest) => 2201 (host) (adapter 1)
==> log: Running 'pre-boot' VM customizations...
==> log: Booting VM...
==> log: Waiting for machine to boot. This may take a few minutes...
    log: SSH address: 127.0.0.1:2201
    log: SSH username: vagrant
    log: SSH auth method: private key
    log:
    log: Vagrant insecure key detected. Vagrant will automatically replace
    log: this with a newly generated keypair for better security.
    log:
    log: Inserting generated public key within guest...
    log: Removing insecure key from the guest if it's present...
    log: Key inserted! Disconnecting and reconnecting using new SSH key...
==> log: Machine booted and ready!
==> log: Checking for guest additions in VM...
    log: No guest additions were detected on the base box for this VM! Guest
    log: additions are required for forwarded ports, shared folders, host only
    log: networking, and more. If SSH fails on this machine, please install
    log: the guest additions and repackage the box to continue.
    log:
    log: This is not an error message; everything may continue to work properly,
    log: in which case you may ignore this message.
==> log: Setting hostname...
==> log: Configuring and enabling network interfaces...
```

Заходим на web-сервер и переходим на пользователя root:

```
neva@Uneva:~$ vagrant ssh web
[vagrant@web ~]$ sudo -i
[root@web ~]#
```

Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время. Укажем часовой пояс (Московское время):

```
[root@web ~]# cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
```

Перезапустим службу NTP Chrony и проверим, что служба работает корректно:

```
[root@web ~]# systemctl restart chronyd
[root@web ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2023-08-01 10:58:23 MSK; 21s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 3001 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 2997 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 2999 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─2999 /usr/sbin/chronyd

Aug 01 10:58:23 web systemd[1]: Stopped NTP client/server.
Aug 01 10:58:23 web systemd[1]: Starting NTP client/server...
Aug 01 10:58:23 web chronyd[2999]: chronyd version 3.4 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCFILTER +SIGND +ASYNCDNS +SECHASH +IPV6 +DEBUG)
Aug 01 10:58:23 web chronyd[2999]: Frequency -1.981 +/- 0.939 ppm read from /var/lib/chrony/drift
Aug 01 10:58:23 web systemd[1]: Started NTP client/server.
Aug 01 10:58:29 web chronyd[2999]: Selected source 192.36.143.130
```

Далее проверим, что время и дата указаны правильно:

```
[root@web ~]# date
Tue Aug  1 11:01:59 MSK 2023
```

Настраиваем время и дату на второй ВМ:

```
neva@Uneva:~$ vagrant ssh log
[vagrant@log ~]$ sudo -i
[root@log ~]# cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
cp: overwrite '/etc/localtime'? y
[root@log ~]# systemctl restart chronyd
[root@log ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2023-08-01 11:19:14 MSK; 6s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 3005 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 3001 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 3003 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─3003 /usr/sbin/chronyd

Aug 01 11:19:14 log systemd[1]: Stopped NTP client/server.
Aug 01 11:19:14 log systemd[1]: Starting NTP client/server...
Aug 01 11:19:14 log chronyd[3003]: chronyd version 3.4 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIVDROP +SCFILTER +SIGND +ASYNCDNS +SECHASH +IPV6 +DEBUG)
Aug 01 11:19:14 log chronyd[3003]: Frequency -2.408 +/- 0.740 ppm read from /var/lib/chrony/drift
Aug 01 11:19:14 log systemd[1]: Started NTP client/server.
Aug 01 11:19:19 log chronyd[3003]: Selected source 162.159.200.1
[root@log ~]# date
Tue Aug  1 11:19:35 MSK 2023
```

2. Установка nginx на виртуальной машине web

Для установки nginx сначала нужно установить epel-release, а затем сам nginx:

```
[vagrant@web ~]$ sudo -i
[root@web ~]# yum install epel-release
[root@web ~]# yum install -y nginx
...
Installed:
  nginx.x86_64 1:1.20.1-10.el7

Dependency Installed:
  centos-indexhtml.noarch 0:7-9.el7.centos        centos-logos.noarch 0:70.0.6-3.el7.centos        gperftools-libs.x86_64 0:2.6.1-1.el7        nginx-filesystem.noarch 1:1.20.1-10.el7        openssl11-libs.x86_64 1:1.1.1k-5.el7
```

Проверим, что nginx работает корректно:

```
[root@web ~]# systemctl start nginx
[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-08-01 11:32:59 MSK; 3s ago
  Process: 3240 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3238 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3237 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3242 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3242 nginx: master process /usr/sbin/nginx
           └─3245 nginx: worker process

Aug 01 11:32:58 web systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 01 11:32:59 web nginx[3238]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 01 11:32:59 web nginx[3238]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 01 11:32:59 web systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@web ~]# ss -tln | grep 80
LISTEN     0      128          *:80                       *:*
LISTEN     0      128       [::]:80                    [::]:*
```

3. Настройка центрального сервера сбора логов

Откроем еще одно окно терминала и подключаемся по ssh к ВМ log, перейдем в пользователя root и проверим, установлен ли rsyslog в нашей ОС:

```
neva@Uneva:~$ vagrant ssh log
Last login: Tue Aug  1 11:14:17 2023 from 10.0.2.2
[vagrant@log ~]$ sudo -i
[root@log ~]# yum list rsyslog
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.checkdomain.de
 * extras: mirror.23m.com
 * updates: ftp.rz.uni-frankfurt.de
base                                                                                                                                                                                                                  | 3.6 kB  00:00:00
extras                                                                                                                                                                                                                | 2.9 kB  00:00:00
updates                                                                                                                                                                                                               | 2.9 kB  00:00:00
(1/4): base/7/x86_64/group_gz                                                                                                                                                                                         | 153 kB  00:00:00
(2/4): extras/7/x86_64/primary_db                                                                                                                                                                                     | 250 kB  00:00:00
(3/4): base/7/x86_64/primary_db                                                                                                                                                                                       | 6.1 MB  00:00:02
(4/4): updates/7/x86_64/primary_db                                                                                                                                                                                    |  22 MB  00:00:07
Installed Packages
rsyslog.x86_64                                                                                                  8.24.0-52.el7                                                                                                       @anaconda
Available Packages
rsyslog.x86_64                                                                                                  8.24.0-57.el7_9.3                                                                                                   updates
```

Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf 
Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в файл: 

Открываем порт 514 (TCP и UDP):
Находим закомментированные строки:

```
# provides UDP syslog reception
#$ModLoad imudp
#$UDPServerRun 514

# provides TCP syslog reception
#$ModLoad imtcp
#$InputTCPServerRun 514
```
И приводим их к виду:

```
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514
```

В конец файла /etc/rsyslog.conf добавляем правила приёма сообщений от хостов:

```
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
```

Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log
Далее сохраняем файл и перезапускаем службу rsyslog: 
Если ошибок не допущено, то у нас будут видны открытые порты TCP,UDP 514:

```
[root@log ~]# nano /etc/rsyslog.conf
[root@log ~]# systemctl restart rsyslog
[root@log ~]# ss -tuln
Netid State      Recv-Q Send-Q                                                                       Local Address:Port                                                                                      Peer Address:Port
udp   UNCONN     0      0                                                                                        *:514                                                                                                  *:*
udp   UNCONN     0      0                                                                                127.0.0.1:323                                                                                                  *:*
udp   UNCONN     0      0                                                                                        *:68                                                                                                   *:*
udp   UNCONN     0      0                                                                                        *:111                                                                                                  *:*
udp   UNCONN     0      0                                                                                        *:927                                                                                                  *:*
udp   UNCONN     0      0                                                                                     [::]:514                                                                                               [::]:*
udp   UNCONN     0      0                                                                                    [::1]:323                                                                                               [::]:*
udp   UNCONN     0      0                                                                                     [::]:111                                                                                               [::]:*
udp   UNCONN     0      0                                                                                     [::]:927                                                                                               [::]:*
tcp   LISTEN     0      25                                                                                       *:514                                                                                                  *:*
tcp   LISTEN     0      128                                                                                      *:111                                                                                                  *:*
tcp   LISTEN     0      128                                                                                      *:22                                                                                                   *:*
tcp   LISTEN     0      100                                                                              127.0.0.1:25                                                                                                   *:*
tcp   LISTEN     0      25                                                                                    [::]:514                                                                                               [::]:*
tcp   LISTEN     0      128                                                                                   [::]:111                                                                                               [::]:*
tcp   LISTEN     0      128                                                                                   [::]:22                                                                                                [::]:*
tcp   LISTEN     0      100                                                                                  [::1]:25                                                                                                [::]:*
```

Далее настроим отправку логов с web-сервера. Проверим версию nginx: 

```
[root@web ~]# rpm -qa | grep nginx
nginx-filesystem-1.20.1-10.el7.noarch
nginx-1.20.1-10.el7.x86_64
```

Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду:

```
error_log /var/log/nginx/error.log;
error_log  syslog:server=192.168.56.15:514,tag=nginx_error;
access_log syslog:server=192.168.56.15:514,tag=nginx_access,severity=info combined
```

Далее проверяем, что конфигурация nginx указана правильно:

```
[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

перезапустим nginx:

```
[root@web ~]# systemctl restart nginx
```

Чтобы проверить, что логи ошибок также улетают на удаленный сервер, можно удалить картинку, к которой будет обращаться nginx во время открытия веб-сраницы: 

```
[root@web ~]# rm /usr/share/nginx/html/img/header-background.png
rm: remove regular file '/usr/share/nginx/html/img/header-background.png'? y
```

Попробуем несколько раз зайти по адресу http://192.168.56.10
Далее заходим на log-сервер и смотрим информацию об nginx:

```
[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Aug  1 13:56:49 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:56:49 +0300] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:56:49 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:56:49 +0300] "GET /img/centos-logo.png HTTP/1.1" 200 3030 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:56:49 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:56:49 +0300] "GET /img/html-background.png HTTP/1.1" 200 1801 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:56:49 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:56:49 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:56:49 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:56:49 +0300] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:57:58 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:57:58 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:57:58 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:57:58 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:57:59 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:57:59 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:57:59 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:57:59 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:00 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:00 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:00 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:00 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:01 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:01 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:01 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:01 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:01 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:01 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:01 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:01 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:01 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:01 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:01 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:01 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Aug  1 13:58:02 web nginx_access: 192.168.56.1 - - [01/Aug/2023:13:58:02 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"

[root@log ~]# cat /var/log/rsyslog/web/nginx_error.log
Aug  1 13:56:49 web nginx_error: 2023/08/01 13:56:49 [error] 22096#22096: *1 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Aug  1 13:56:49 web nginx_error: 2023/08/01 13:56:49 [error] 22096#22096: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Aug  1 13:57:58 web nginx_error: 2023/08/01 13:57:58 [error] 22096#22096: *4 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Aug  1 13:57:59 web nginx_error: 2023/08/01 13:57:59 [error] 22096#22096: *4 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Aug  1 13:58:00 web nginx_error: 2023/08/01 13:58:00 [error] 22096#22096: *4 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
```

4. Настройка аудита, контролирующего изменения конфигурации nginx

За аудит отвечает утилита auditd, в RHEL-based системах обычно он уже предустановлен. Проверим это: 

```
[root@web ~]# rpm -qa | grep audit
audit-2.8.5-4.el7.x86_64
audit-libs-2.8.5-4.el7.x86_64
```

Настроим аудит изменения конфигурации nginx:
Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого в конец файла /etc/audit/rules.d/audit.rules добавим следующие строки:

```
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
```
Данные правила позволяют контролировать запись (w) и измения атрибутов (a) в:
●	/etc/nginx/nginx.conf
●	Всех файлов каталога /etc/nginx/default.d/
Для более удобного поиска к событиям добавляется метка nginx_conf

Перезапускаем службу auditd:

```
[root@web ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
```

После данных изменений у нас начнут локально записываться логи аудита. Чтобы это проверить, внесём  изменения в файл /etc/nginx/nginx.conf и посмотрим информацию об изменениях:

```
[root@web ~]# ausearch -f /etc/nginx/nginx.conf
----
time->Wed Aug  2 15:18:00 2023
type=PROCTITLE msg=audit(1690978680.861:1126): proctitle=6E616E6F002F6574632F6E67696E782F6E67696E782E636F6E66
type=PATH msg=audit(1690978680.861:1126): item=1 name="/etc/nginx/nginx.conf" inode=33577660 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=PATH msg=audit(1690978680.861:1126): item=0 name="/etc/nginx/" inode=33577648 dev=08:01 mode=040755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=CWD msg=audit(1690978680.861:1126):  cwd="/root"
type=SYSCALL msg=audit(1690978680.861:1126): arch=c000003e syscall=2 success=yes exit=3 a0=1345840 a1=441 a2=1b6 a3=63 items=2 ppid=22850 pid=22886 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="nano" exe="/usr/bin/nano" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
----
time->Wed Aug  2 15:18:43 2023
type=PROCTITLE msg=audit(1690978723.339:1127): proctitle=6E616E6F002F6574632F6E67696E782F6E67696E782E636F6E66
type=PATH msg=audit(1690978723.339:1127): item=1 name="/etc/nginx/nginx.conf" inode=33577660 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=PATH msg=audit(1690978723.339:1127): item=0 name="/etc/nginx/" inode=33577648 dev=08:01 mode=040755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=CWD msg=audit(1690978723.339:1127):  cwd="/root"
type=SYSCALL msg=audit(1690978723.339:1127): arch=c000003e syscall=2 success=yes exit=3 a0=1348700 a1=241 a2=1b6 a3=7ffc0740e1a0 items=2 ppid=22850 pid=22886 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="nano" exe="/usr/bin/nano" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
```

Настроим пересылку логов на удаленный сервер. Auditd по умолчанию не умеет пересылать логи, для пересылки на web-сервере потребуется установить пакет audispd-plugins: 

```
yum -y install audispd-plugins
Installed:
  audispd-plugins.x86_64 0:2.8.5-4.el7

Complete!
```
Найдем и поменяем следующие строки в файле /etc/audit/auditd.conf:

```
log_format = RAW
name_format = HOSTNAME
```
В name_format  указываем HOSTNAME, чтобы в логах на удаленном сервере отображалось имя хоста. 

В файле /etc/audisp/plugins.d/au-remote.conf поменяем параметр active на yes:

```
active = yes
direction = out
path = /sbin/audisp-remote
type = always
#args =
format = string
```

В файле /etc/audisp/audisp-remote.conf требуется указать адрес сервера и порт, на который будут отправляться логи:

```
remote_server = 192.168.56.15
port = 60
```

перезапускаем службу auditd: 

```
[root@web ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
```

На этом настройка web-сервера завершена. Далее настроим Log-сервер. Отроем порт TCP 60, для этого уберем значки комментария в файле /etc/audit/auditd.conf:

```
tcp_listen_port = 60
```

Перезапустим службу auditd:

```
[root@log ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
```

На этом настройка пересылки логов аудита закончена. Можем попробовать поменять атрибут у файла /etc/nginx/nginx.conf и проверить на log-сервере, что пришла информация об изменении атрибута:

```
[root@web ~]# ls -l /etc/nginx/nginx.conf
-rw-r--r--. 1 root root 2566 Aug  2 15:18 /etc/nginx/nginx.conf
[root@web ~]# chmod +x /etc/nginx/nginx.conf
[root@web ~]# ls -l /etc/nginx/nginx.conf
-rwxr-xr-x. 1 root root 2566 Aug  2 15:18 /etc/nginx/nginx.conf

[root@log ~]# grep web /var/log/audit/audit.log
node=web type=DAEMON_START msg=audit(1690979792.103:4683): op=start ver=2.8.5 format=raw kernel=3.10.0-1127.el7.x86_64 auid=4294967295 pid=23000 uid=0 ses=4294967295 subj=system_u:system_r:auditd_t:s0 res=success
node=web type=CONFIG_CHANGE msg=audit(1690979792.258:1133): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1690979792.260:1134): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1690979792.260:1135): audit_backlog_limit=8192 old=8192 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1690979792.260:1136): audit_failure=1 old=1 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1690979792.261:1137): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1690979792.261:1138): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SERVICE_START msg=audit(1690979792.261:1139): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=auditd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
node=web type=SYSCALL msg=audit(1690980219.110:1140): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=21f1420 a2=1ed a3=7fffed9212e0 items=1 ppid=22850 pid=23025 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CWD msg=audit(1690980219.110:1140):  cwd="/root"
node=web type=PATH msg=audit(1690980219.110:1140): item=0 name="/etc/nginx/nginx.conf" inode=33577660 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
node=web type=PROCTITLE msg=audit(1690980219.110:1140): proctitle=63686D6F64002B78002F6574632F6E67696E782F6E67696E782E636F6E66
```





