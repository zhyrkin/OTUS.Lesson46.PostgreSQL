# OTUS.Lesson46.PostgreSQL
Описание домашнего задания
1) Настроить hot_standby репликацию с использованием слотов
2) Настроить правильное резервное копирование

Выполнение домашнего задания:
Тестовый стенд 2VM (2CPU, 2GB RAM, 6Gb HDD, OS Debain 12)
1) Запускаем плейбук postrgres_replica.yml и ждем выполнения
На node 1 создадим базу и убедимся что она появилась на node2:
```
postgres@node1:/root$ psql 
could not change directory to "/root": Отказано в доступе
psql (15.12 (Debian 15.12-0+deb12u2))
Type "help" for help.

postgres=# 
postgres=# CREATE DATABASE otus_test;
CREATE DATABASE
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 otus_test | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 postgres  | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)
postgres=# select * from pg_stat_replication;
  pid  | usesysid |   usename   | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time           
-------+----------+-------------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+------------+---------------+------------+-------------------------------
 12440 |    16384 | replication | walreceiver      | 10.200.3.96 |                 |       54640 | 2025-05-07 03:18:45.324168-03 |          737 | streaming | 0/5422140 | 0/5422140 | 0/5422140 | 0/5422140  |           |           |            |             0 | async      | 2025-05-07 03:32:59.254563-03
(1 row)
```
проверяем на node2
```
root@node2:~# su postgres 
postgres@node2:/root$ psql 
could not change directory to "/root": Отказано в доступе
psql (15.12 (Debian 15.12-0+deb12u2))
Type "help" for help.

postgres=# 
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 otus_test | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 postgres  | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)

postgres=#  select * from pg_stat_wal_receiver;
  pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time      |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        | slot_name | sender_host | sender_port |                                                                                                                                        conninfo                                                                                                                                        
-------+-----------+-------------------+-------------------+-------------+-------------+--------------+------------------------------+-------------------------------+----------------+-------------------------------+-----------+-------------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 10843 | streaming | 0/5000000         |                 1 | 0/5422140   | 0/5000000   |            1 | 2025-05-07 03:41:31.25277-03 | 2025-05-07 03:41:31.253252-03 | 0/5422140      | 2025-05-07 03:40:31.085827-03 |           | 10.200.3.95 |        5432 | user=replication password=******** channel_binding=prefer dbname=replication host=10.200.3.95 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
(1 row)
```
2) Запускаем playbook postgresql_barman.yml и ждем выполнения. 
Проверяем возможность подключения с сервера barman к node1:
```
barman@barman:~$ psql -h 10.200.3.95 -U barman -d postgres 
psql (15.12 (Debian 15.12-0+deb12u2))
Введите "help", чтобы получить справку.

postgres=# 
```

На VM barman
```
root@barman:~# barman switch-wal node1
The WAL file 000000010000000000000004 has been closed on server 'node1'
root@barman:~#  barman cron
Starting WAL archiving for server node1
root@barman:~#  barman check node1
Server node1:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
	backup minimum size: OK (0 B)
	wal maximum age: OK (no last_wal_maximum_age provided)
	wal size: OK (0 B)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archiver errors: OK
```
Запускаем процедуру бекапа:
```
root@barman:~# barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20250507T142220
Backup start at LSN: 0/6000060 (000000010000000000000006, 00000060)
Starting backup copy via pg_basebackup for 20250507T142220
WARNING: pg_basebackup does not copy the PostgreSQL configuration files that reside outside PGDATA. Please manually backup the following files:
	/etc/postgresql/15/main/postgresql.conf
	/etc/postgresql/15/main/pg_hba.conf
	/etc/postgresql/15/main/pg_ident.conf

Copy done (time: 4 seconds)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
	000000010000000000000004 from server node1 has been removed
	000000010000000000000005 from server node1 has been removed
Backup size: 29.3 MiB
Backup end at LSN: 0/8000000 (000000010000000000000007, 00000000)
Backup completed (start time: 2025-05-07 14:22:20.890742, elapsed time: 5 seconds)
Processing xlog segments from streaming for node1
	000000010000000000000006
	000000010000000000000007
```
Проверим восстановление базы из бекапа.
На node1 удалим  базу otus и otus_test:
```
postgres=# DROP DATABASE otus;
DROP DATABASE
postgres=# DROP DATABASE otus_test; 
DROP DATABASE
postgres=# \l
                                                  Список баз данных
    Имя    | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   | локаль ICU | Провайдер локали |     Права доступа     
-----------+----------+-----------+-------------+-------------+------------+------------------+-----------------------
 postgres  | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 template0 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/postgres          +
           |          |           |             |             |            |                  | postgres=CTc/postgres
 template1 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/postgres          +
           |          |           |             |             |            |                  | postgres=CTc/postgres
(3 строки)
```
Далее на хосте barman запустим восстановление: 
```
root@barman:~# barman list-backup node1
node1 20250507T144459 - Wed May  7 08:45:03 2025 - Size: 36.5 MiB - WAL Size: 0 B
root@barman:~# barman recover node1  20250507T144459 /var/lib/postgresql/15/main/ --remote-ssh-comman "ssh postgres@10.200.3.95"
The authenticity of host '10.200.3.95 (10.200.3.95)' can't be established.
ED25519 key fingerprint is SHA256:R7XAEox3a3M00Obxnxz+80TVzrmQkLIhqmJGmAFKgjk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20250507T144459
Destination directory: /var/lib/postgresql/15/main/
Remote command: ssh postgres@10.200.3.95
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

WARNING
The following configuration files have not been saved during backup, hence they have not been restored.
You need to manually restore them in order to start the recovered PostgreSQL instance:

    postgresql.conf
    pg_hba.conf
    pg_ident.conf

Recovery completed (start time: 2025-05-07 14:47:50.180426+03:00, elapsed time: 8 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```
НА node1 перезапускаем postgresql и убедимся что базульки восстановлены:
```
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 otus      | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 otus_test | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 postgres  | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | 
 template0 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(5 rows)

```
