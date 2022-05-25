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

