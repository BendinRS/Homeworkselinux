# Домашняя работа "Стенд с Vagrant c SELinux"

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

                    У - успех
![Альтернативный текст](https://i.ibb.co/kQvq8BM/nginx.png)






