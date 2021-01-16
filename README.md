# 1. Запускаем nginx на нестандартных портах

## Вариант 1. Меняем порт 80 в файле /etc/nginx/nginx.conf на 5080.

Рестарт не проходит:

```
[root@localhost nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Сб 2021-01-16 12:33:15 UTC; 20s ago
  Process: 12923 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 12922 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

янв 16 12:33:15 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
янв 16 12:33:15 localhost.localdomain nginx[12923]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
янв 16 12:33:15 localhost.localdomain nginx[12923]: nginx: [emerg] bind() to 0.0.0.0:5080 failed (13: Permission denied)
янв 16 12:33:15 localhost.localdomain nginx[12923]: nginx: configuration file /etc/nginx/nginx.conf test failed
янв 16 12:33:15 localhost.localdomain systemd[1]: nginx.service: control process exited, code=exited status=1
янв 16 12:33:15 localhost.localdomain systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
янв 16 12:33:15 localhost.localdomain systemd[1]: Unit nginx.service entered failed state.
янв 16 12:33:15 localhost.localdomain systemd[1]: nginx.service failed.
[root@localhost nginx]# 
```

Делаем

```
[root@localhost nginx]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1610800395.876:924): avc:  denied  { name_bind } for  pid=12923 comm="nginx" src=5080 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@localhost nginx]# 
```
```
[root@localhost nginx]# setsebool -P nis_enabled 1
[root@localhost nginx]# systemctl start nginx
[root@localhost nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Сб 2021-01-16 12:36:32 UTC; 7s ago
  Process: 12947 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 12944 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 12943 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 12949 (nginx)
   CGroup: /system.slice/nginx.service
           ├─12949 nginx: master process /usr/sbin/nginx
           └─12950 nginx: worker process

янв 16 12:36:32 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
янв 16 12:36:32 localhost.localdomain nginx[12944]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
янв 16 12:36:32 localhost.localdomain nginx[12944]: nginx: configuration file /etc/nginx/nginx.conf test is successful
янв 16 12:36:32 localhost.localdomain systemd[1]: Failed to parse PID from file /run/nginx.pid: Invalid argument
янв 16 12:36:32 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@localhost nginx]# 
```
```
[root@localhost nginx]# netstat -lntp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      337/rpcbind         
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      675/sshd            
tcp        0      0 0.0.0.0:5080            0.0.0.0:*               LISTEN      12949/nginx: master 
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      909/master          
tcp6       0      0 :::111                  :::*                    LISTEN      337/rpcbind         
tcp6       0      0 :::80                   :::*                    LISTEN      12949/nginx: master 
tcp6       0      0 :::22                   :::*                    LISTEN      675/sshd            
tcp6       0      0 ::1:25                  :::*                    LISTEN      909/master          
[root@localhost nginx]# 
```

Теперь выключаем 
```
[root@localhost nginx]# setsebool -P nis_enabled 0
```
И nginx уже не запускается.

## Вариант 2. Добавление нестандартного порта к имеющимся.

```
[root@localhost nginx]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@localhost nginx]# semanage port -a -t http_port_t -p tcp 5080
[root@localhost nginx]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      5080, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@localhost nginx]# systemctl start nginx
[root@localhost nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Сб 2021-01-16 12:52:26 UTC; 4s ago
  Process: 13307 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 13304 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 13303 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 13309 (nginx)
   CGroup: /system.slice/nginx.service
           ├─13309 nginx: master process /usr/sbin/nginx
           └─13310 nginx: worker process

янв 16 12:52:26 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
янв 16 12:52:26 localhost.localdomain nginx[13304]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
янв 16 12:52:26 localhost.localdomain nginx[13304]: nginx: configuration file /etc/nginx/nginx.conf test is successful
янв 16 12:52:26 localhost.localdomain systemd[1]: Failed to parse PID from file /run/nginx.pid: Invalid argument
янв 16 12:52:26 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@localhost nginx]# semanage port -d -t http_port_t -p tcp 5080
[root@localhost nginx]# systemctl stop nginx
[root@localhost nginx]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@localhost nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Сб 2021-01-16 12:53:12 UTC; 7s ago
  Process: 13333 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 13332 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

янв 16 12:53:12 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
янв 16 12:53:12 localhost.localdomain nginx[13333]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
янв 16 12:53:12 localhost.localdomain nginx[13333]: nginx: [emerg] bind() to 0.0.0.0:5080 failed (13: Permission denied)
янв 16 12:53:12 localhost.localdomain nginx[13333]: nginx: configuration file /etc/nginx/nginx.conf test failed
янв 16 12:53:12 localhost.localdomain systemd[1]: nginx.service: control process exited, code=exited status=1
янв 16 12:53:12 localhost.localdomain systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
янв 16 12:53:12 localhost.localdomain systemd[1]: Unit nginx.service entered failed state.
янв 16 12:53:12 localhost.localdomain systemd[1]: nginx.service failed.
[root@localhost nginx]# 
```

## Вариант 3. Формирование и установка модуля SELinux.

```
[root@localhost nginx]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_add.pp

[root@localhost nginx]# semodule -i httpd_add.pp
[root@localhost nginx]# semodule -l | grep http
httpd_add	1.0
[root@localhost nginx]# systemctl start nginx
[root@localhost nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Сб 2021-01-16 13:10:03 UTC; 3s ago
  Process: 13378 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 13376 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 13375 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 13380 (nginx)
   CGroup: /system.slice/nginx.service
           ├─13380 nginx: master process /usr/sbin/nginx
           └─13381 nginx: worker process

янв 16 13:10:03 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
янв 16 13:10:03 localhost.localdomain nginx[13376]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
янв 16 13:10:03 localhost.localdomain nginx[13376]: nginx: configuration file /etc/nginx/nginx.conf test is successful
янв 16 13:10:03 localhost.localdomain systemd[1]: Failed to parse PID from file /run/nginx.pid: Invalid argument
янв 16 13:10:03 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@localhost nginx]# 
```



# 2. Обеспечить работоспособность приложения при включенном selinux.

```
me@me-HP-260-G3-DM:~/otus-linux/DZ20210116-2/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
me@me-HP-260-G3-DM:~/otus-linux/DZ20210116-2/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Sat Jan 16 15:03:46 2021 from 10.0.2.2
```

Дальше вывод с сервера после попыток обновить зону с клиента:

```
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1610809696.688:1872): avc:  denied  { create } for  pid=4973 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# audit2allow -M named-selinux --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i named-selinux.pp

[root@ns01 ~]# semodule -i named-selinux.pp
[root@ns01 ~]# 
```


Ошибки остались, делаем ещё один модуль:

```
[root@ns01 ~]# cat /var/log/messages | grep ausearch
Jan 16 15:08:20 localhost python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
[root@ns01 ~]# cat /var/log/messages | grep ausearch
Jan 16 15:08:20 localhost python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
Jan 16 15:13:28 localhost python: SELinux is preventing isc-worker0000 from write access on the file /etc/named/dynamic/named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have write access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on /etc/named/dynamic/named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE '/etc/named/dynamic/named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: afs_cache_t, dnssec_trigger_var_run_t, initrc_tmp_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t, puppet_tmp_t, user_cron_spool_t, user_tmp_t.#012Then execute:#012restorecon -v '/etc/named/dynamic/named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed write access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
[root@ns01 ~]# 

[root@ns01 ~]# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker000
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-iscworker000.pp

[root@ns01 ~]# semodule -i my-iscworker000.pp
[root@ns01 ~]# 
```

Теперь смотрим на статус сервиса, удаляем проблемный лог-файл и перезагрузаем сервис:

```
[root@ns01 ~]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2021-01-16 15:03:47 UTC; 16min ago
  Process: 4971 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 4969 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 4973 (named)
   CGroup: /system.slice/named.service
           └─4973 /usr/sbin/named -u named -c /etc/named.conf

Jan 16 15:13:24 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#55780/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Jan 16 15:13:24 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#55780/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': add....168.50.15
Jan 16 15:13:24 ns01 named[4973]: /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied
Jan 16 15:13:24 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#55780/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': err...cted error
Jan 16 15:17:58 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#55780/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Jan 16 15:17:58 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#55780/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': add....168.50.15
Jan 16 15:17:58 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#55780/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': err...d: no more
Jan 16 15:19:19 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#41160/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Jan 16 15:19:19 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#41160/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': add....168.50.15
Jan 16 15:19:19 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#41160/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': err...d: no more
Hint: Some lines were ellipsized, use -l to show in full.
[root@ns01 ~]# 

[root@ns01 ~]# rm /etc/named/dynamic/named.ddns.lab.view1.jnl
rm: remove regular empty file '/etc/named/dynamic/named.ddns.lab.view1.jnl'? y
[root@ns01 ~]# systemctl reload named
[root@ns01 ~]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2021-01-16 15:03:47 UTC; 19min ago
  Process: 5101 ExecReload=/bin/sh -c /usr/sbin/rndc reload > /dev/null 2>&1 || /bin/kill -HUP $MAINPID (code=exited, status=0/SUCCESS)
  Process: 4971 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 4969 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 4973 (named)
   CGroup: /system.slice/named.service
           └─4973 /usr/sbin/named -u named -c /etc/named.conf

Jan 16 15:22:31 ns01 named[4973]: network unreachable resolving './DNSKEY/IN': 2001:7fd::1#53
Jan 16 15:22:31 ns01 named[4973]: network unreachable resolving './DNSKEY/IN': 2001:500:2f::f#53
Jan 16 15:22:31 ns01 named[4973]: network unreachable resolving './DNSKEY/IN': 2001:500:9f::42#53
Jan 16 15:22:31 ns01 named[4973]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
Jan 16 15:22:31 ns01 named[4973]: all zones loaded
Jan 16 15:22:31 ns01 named[4973]: running
Jan 16 15:22:32 ns01 named[4973]: managed-keys-zone/view1: Key 20326 for zone . acceptance timer complete: key now trusted
Jan 16 15:22:32 ns01 named[4973]: managed-keys-zone/default: Key 20326 for zone . acceptance timer complete: key now trusted
Jan 16 15:23:03 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#38258/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Jan 16 15:23:03 ns01 named[4973]: client @0x7f182c03c3e0 192.168.50.15#38258/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'w...2.168.50.15
Hint: Some lines were ellipsized, use -l to show in full.
[root@ns01 ~]# 
```

С клиента получилось обновить зону:

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> 
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> send
> exit
incorrect section name: exit
> quit
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> ыутв
incorrect section name: ыутв
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> 
```

