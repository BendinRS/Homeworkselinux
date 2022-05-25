# Домашняя работа "Стенд с Vagrant c SELinux"

<details>
  <summary>Запустить nginx на нестандартном порту 3-мя разными способами </summary>

### Цель: Диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений

1. Запустить nginx на нестандартном порту 3-мя разными способами:
    + переключатели setsebool;
    + добавление нестандартного порта в имеющийся тип;
    + формирование и установка модуля SELinux.

+ Создаем вм [vagrantfile](vagrantfile) 

В ходе загрузки ВМ пытается запустится nginx на порту 4881, но завершается с ошибкой
            Вывод systemctl status nginx
![Альтернативный текст](https://i.ibb.co/2dh3J6J/selinux-fail1.png)

#### переключатели setsebool

+ Проверяем, отключен ли фаервол:
```out
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
    Docs: man:firewalld(1)
```
+ Проверяем, что конфигурация nginx построена без ошибок
```out1
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
+ Проверяем режим работы selinux
```out2
[root@selinux ~]# getenforce 
Enforcing
```
**Enforcing** - Активная работа. Всё, что нарушает
политику безопасности блокируется. Попытка нарушения фиксируется
в журнале.

+ Находим в логах информацию о блокировании порта
![Альтернативный текст](https://i.ibb.co/LJdgTgh/selinfail2.png)

+ Устанавливаем audit2why
```out2
[root@selinux ~]# yum install policycoreutils-python
```

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим информации о запрете

```out3
[root@selinux ~]# grep 1653503853.097:788 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1653503853.097:788): avc:  denied  { name_bind } for  pid=2760 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

audit2why подсказывает нам, что необходимо изменить параметр nis_enabled

 ***У - успех***
![Альтернативный текст](https://i.ibb.co/kQvq8BM/nginx.png)


#### Добавление нестандартного порта в имеющийся тип

+ Отключаем nis_enabled, что бы снова вернуть запрет работы nginx

```out5
[root@selinux ~]# setsebool -P nis_enabled off
```

+ Ищем имеющийс тип для http трафика

```out6
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

+ Добавляем порт 4881

```out7
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

**У-успех**
![Альтернативный текст](https://i.ibb.co/Y3yD7CD/nginx2.png)


#### Формирование и установка модуля SELinux

+ Удаляем нестандартный порт из имеющегося типа

```out8
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

+ попытка запустить nginx
```out9
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

+ Идем в логи

```out10
type=SYSCALL msg=audit(1653507918.442:917): arch=c000003e syscall=49 success=no exit=-13 a0=7 a1=562223f307d8 a2=1c a3=7ffdec16e4b4 items=0 ppid=1 pid=22039 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1653507918.446:918): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```

+ Воспользуемся утилитой audit2allow для того, чтобы на основе логов seLinux сделать модуль, разрешающий работу nginx на нестандартном порту

```out11
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

+ Применяем сформированный модуль

__У-успех__
![Альтернативный текст](https://i.ibb.co/gJb448h/nginx3.png)
</details>

<details>
  <summary> Обеспечение работоспособности приложения при включенном SELinux </summary>
   
 + Развернем предложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
![Альтернативный текст](https://i.ibb.co/Mfn5CkV/sel2.png)

+ Подключаемся к клиенту и пробуем внести изменения

```out20
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

+ Переходим в логи

на клиенте отсутствуют ошибки.

+ Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи selinux

```out21
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1653510348.341:1863): avc:  denied  { create } for  pid=5009 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# 
```
Ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.

+ Проверяем каталог /etc/named

```out22
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге.
Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: 
```out 23
sudo semanage fcontext -l | grep named ```
```

+ Изменяем тип контекста безопасности

```out24
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]#  ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

+ Вносим изменения с клиента и проверяем
![Альтернативный текст](https://i.ibb.co/fvxg7Ww/screen.png)





</details>