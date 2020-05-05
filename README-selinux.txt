- ПЕРЕКЛЮЧАТЕЛИ setsebool;

1. Проверяем работу nginx и порт который он слушает в данный момент

[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2020-05-05 06:09:22 UTC; 2min 39s ago
  Process: 5078 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 5075 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 5074 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 5080 (nginx)
   CGroup: /system.slice/nginx.service
           ├─5080 nginx: master process /usr/sbin/nginx
           └─5081 nginx: worker process

May 05 06:09:22 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 05 06:09:22 localhost.localdomain nginx[5075]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 05 06:09:22 localhost.localdomain nginx[5075]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 05 06:09:22 localhost.localdomain systemd[1]: Failed to read PID from file /run/nginx.pid: Invalid argument
May 05 06:09:22 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.

[root@localhost ~]# netstat -ntlp | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      5080/nginx: master  

2. Меняем порт на нестандартный(25255) в nginx.conf

3. Перезапускаем nginx, получаем ошибку:

[root@localhost ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

[root@localhost ~]# journalctl -xe

...

-- Unit nginx.service has begun starting up.
May 05 06:14:05 localhost.localdomain nginx[26102]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 05 06:14:05 localhost.localdomain nginx[26102]: nginx: [emerg] bind() to 0.0.0.0:25255 failed (13: Permission denied)
May 05 06:14:05 localhost.localdomain nginx[26102]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 05 06:14:05 localhost.localdomain systemd[1]: nginx.service: control process exited, code=exited status=1
May 05 06:14:05 localhost.localdomain systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
-- Subject: Unit nginx.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel

...


4. Смотрим лог при помощи утилиты audit2why:

[root@localhost ~]# audit2why < /var/log/audit/audit.log 
type=AVC msg=audit(1588658684.781:895): avc:  denied  { name_bind } for  pid=4762 comm="nginx" src=25255 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
type=AVC msg=audit(1588659245.251:1064): avc:  denied  { name_bind } for  pid=26102 comm="nginx" src=25255 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1

5. Изменяем значение политик следуя рекомендациям лога

[root@localhost ~]# setsebool -P nis_enabled 1

6. Запускаем nginx и проверяем слушает ли он указанный нами нестандартный порт:

[root@localhost ~]# systemctl start nginx
[root@localhost ~]# netstat -ntlp | grep nginx
tcp        0      0 0.0.0.0:25255           0.0.0.0:*               LISTEN      26132/nginx: master 
tcp6       0      0 :::80                   :::*                    LISTEN      26132/nginx: master 

7. Останаливаем nginx и меняем значение политики на 0:
[root@localhost ~]# systemctl stop nginx
[root@localhost ~]# setsebool -P nis_enabled 0


- ДОБАВЛЕНИЕ НЕСТАНДАРТНОГО ПОРТА В ИМЕЮЩИЙСЯ ТИП


1. Проверяем перечень разрешенных портов в имеющемся типе:

[root@localhost ~]# semanage port -l | grep http_port
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000

2. Добавляем "наш" порт в этот перчень:
[root@localhost ~]# semanage port -a -t http_port_t -p tcp 25255

3. Проверяем:

[root@localhost ~]# semanage port -l | grep http_port
http_port_t                    tcp      25255, 80, 81, 443, 488, 8008, 8009, 8443, 9000

4. Запускаем nginx и проверяем слушает ли он указанный нами нестандартный порт:

[root@localhost ~]# systemctl start nginx
[root@localhost ~]# netstat -ntlp | grep nginx
tcp        0      0 0.0.0.0:25255           0.0.0.0:*               LISTEN      26209/nginx: master 

5. Удаляем "наш" порт из перечня и останавливаем nginx:

[root@localhost ~]# semanage port -d -t http_port_t -p tcp 25255
[root@localhost ~]# semanage port -l | grep http_port
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@localhost ~]# systemctl stop nginx


- ФОРМИРОВАНИЕ И УСТАНОВКА МОДУЛЯ SELinux.

1. Очищаем audit.log

[root@localhost ~]# cp /dev/null /var/log/audit/audit.log

2. Пытаемся запустить nginx:

[root@localhost ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

3. Формирует модуль с правилами для SELinux:

[root@localhost ~]# audit2allow -M nginx_add --debug < /var/log/audit/audit.log

4. Загружаем модуль и запускаем nginx:

[root@localhost ~]# semodule -i nginx_add.pp
[root@localhost ~]# systemctl start nginx
[root@localhost ~]# netstat -ntlp | grep nginx
tcp        0      0 0.0.0.0:25255           0.0.0.0:*               LISTEN      26348/nginx: master

5. Удалям модуль

[root@localhost ~]# semodule -r nginx_add



