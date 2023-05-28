## Lesson17 - SELINUX

1. Запустить nginx на нестандартном порту 3-мя разными способами:  
* переключатели setsebool;  
* добавление нестандартного порта в имеющийся тип;  
* формирование и установка модуля SELinux.  
* К сдаче:  
* README с описанием каждого решения (скриншоты и демонстрация приветствуются).  

2. Обеспечить работоспособность приложения при включенном selinux.  
* развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;  
* выяснить причину неработоспособности механизма обновления зоны (см. README);  
* предложить решение (или решения) для данной проблемы;  
* выбрать одно из решений для реализации, предварительно обосновав выбор;  
* реализовать выбранное решение и продемонстрировать его работоспособность.  
* К сдаче:
* README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;  
* исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.  


### Полезные данные для работы:

Режим по умолчанию - /etc/sysconfig/selinux

Файлы политик размещаются по пути - /etc/selinux/targeted/contexts/files  

Лог аудита храниться в файле /var/log/audit/audit.log

Просмотреть текущие правила - sesearch -A -s httpd_t

Правильно задать контекст ветвей файловой системы необходимо через semanage

```
semanage fcontext -a -t httpd_sys_content_t "/html(/.*)?"

```

audit2why - преобразовывает сообщения аудита SELinux в описание причины отказа  в  доступе (audit2allow -w)  

audit2allow  -  создаёт  правила  политики SELinux allow/dontaudit из журналов отклонённых операций.
При создании модуля создается файл .te (Type Enforcement) и файл .pp (скомпилированный пакет политики).

```
audit2allow -M httpd_add --debug < /var/log/audit/audit.log
```

Включение созданного модуля 
```
semodule -i httpd_add.pp. Убедиться что модуль подключен - semodule -l | grep httpd_add
```

Удаление модуля
```
semodule -r httpd_add
```



### 1. Запустить nginx на нестандартном порту 3-мя разными способами:  


#### Метод 1 (setsebool)

Запускаем Vagrantfile и видим, что nginx не запустился.

```
linux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Sat 2023-01-21 19:09:34 UTC; 9ms ago
    selinux:   Process: 2787 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2786 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux: 
    selinux: Jan 21 19:09:34 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Jan 21 19:09:34 selinux nginx[2787]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Jan 21 19:09:34 selinux nginx[2787]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Jan 21 19:09:34 selinux nginx[2787]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Jan 21 19:09:34 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Jan 21 19:09:34 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Jan 21 19:09:34 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Jan 21 19:09:34 selinux systemd[1]: nginx.service failed.
```

Запускаем audit2why  и анализируем лог:
```
[root@selinux ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1674328174.938:800): avc:  denied  { name_bind } for  pid=2787 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

Выполняем вышепредложенную рекомендацию
```
setsebool -P nis_enabled 1
```

Проверяем, что запустилось.
```
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2023-01-21 19:31:24 UTC; 1s ago
  Process: 2958 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2956 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2955 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2960 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2960 nginx: master process /usr/sbin/nginx
           └─2962 nginx: worker process
```

#### Метод 2 (setsebool)

Смотрим какие порты могут работать по http(видим, что нашего порта нет в списке):

```
[root@selinux ~]# semanage port -l | grep htt
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавляем порт из конфига nginx и проверяем:

```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# systemctl restart nginx.service 
[root@selinux ~]# semanage port -l | grep htt
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux ~]# systemctl status nginx.service 
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2023-01-21 19:48:46 UTC; 28s ago
  Process: 2902 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2900 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2899 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2904 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2904 nginx: master process /usr/sbin/nginx
           └─2906 nginx: worker process

```

#### Метод 3 (формирование и установка модуля SELinux)

Скомпилируем модуль на основе лог файла аудита с инфой о запретах:

```
[root@selinux ~]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_add.pp
```
Устанавливаем модуль и проверяем:

```
[root@selinux ~]# semodule -i httpd_add.pp
[root@selinux ~]# semodule -l | grep http
httpd_add	1.0
Failed to restart ngix.service: Unit not found.
[root@selinux ~]# systemctl restart nginx.service
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2023-01-21 20:13:04 UTC; 5s ago
  Process: 2956 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2954 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2953 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2958 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2958 nginx: master process /usr/sbin/nginx
           └─2960 nginx: worker process

Jan 21 20:13:04 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 21 20:13:04 selinux nginx[2954]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 21 20:13:04 selinux nginx[2954]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 21 20:13:04 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```


2. Обеспечить работоспособность приложения при включенном selinux.  

Подключаемся к клиенту и пробуем внести изменеия в зону:
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```


Видим, что на клиенте отсутствуют ошибки.

```
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```

Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:

```
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1674333563.659:1888): avc:  denied  { create } for  pid=5085 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

```

В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. Проверим данную проблему в каталоге /etc/named.

```
[root@ns01 ~]# ls -Zla /etc/named/*
-rw-rw----. 1 system_u:object_r:etc_t:s0       root named 784 Jan 21 20:31 /etc/named/named.50.168.192.rev
-rw-rw----. 1 system_u:object_r:etc_t:s0       root named 610 Jan 21 20:30 /etc/named/named.dns.lab
-rw-rw----. 1 system_u:object_r:etc_t:s0       root named 609 Jan 21 20:30 /etc/named/named.dns.lab.view1
-rw-rw----. 1 system_u:object_r:etc_t:s0       root named 657 Jan 21 20:31 /etc/named/named.newdns.lab

/etc/named/dynamic:
total 8
drw-rwx---. 2 unconfined_u:object_r:etc_t:s0   root  named  56 Jan 21 20:31 .
drw-rwx---. 3 system_u:object_r:etc_t:s0       root  named 121 Jan 21 20:31 ..
-rw-rw----. 1 system_u:object_r:etc_t:s0       named named 509 Jan 21 20:30 named.ddns.lab
-rw-rw----. 1 system_u:object_r:etc_t:s0       named named 509 Jan 21 20:31 named.ddns.lab.view1
[root@ns01 ~]# 

```

Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:

```
sudo semanage fcontext -l | grep named
```

Изменим тип контекста безопасности для каталога /etc/named:

```
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab

```

Попробуем снова внести изменения с клиента и проверяем:

```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11363
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Jan 21 21:16:55 UTC 2023
;; MSG SIZE  rcvd: 96

```
Работает!


Проблема с обновлением зоны заключалась в том, что Selinux блокировал доступ к обновлению файлов динамического обновления для DNS сервера
