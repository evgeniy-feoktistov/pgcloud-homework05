# pgcloud-homework05

# Домашнее задание
## Бэкапы Постгреса

**Цель**:
Используем современные решения для бэкапов

Описание/Пошаговая инструкция выполнения домашнего задания:
Делам бэкап Постгреса используя WAL-G или pg_probackup и восстанавливаемся на другом кластере.

Установка pg_probackup
```bash
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
rpm -ivh https://repo.postgrespro.ru/pg_probackup/keys/pg_probackup-repo-centos.noarch.rpm
yum install pg_probackup-11
```

Создадим каталог для бэкапов
```bash
root@psql01:/home/ujack# sudo mkdir /home/backups && sudo chmod 777 /home/backups
```

Пропишем переменную окружения
```bash
root@psql01:/home/ujack# echo "BACKUP_PATH=/home/backups/">>~/.bashrc
root@psql01:/home/ujack# echo "export BACKUP_PATH">>~/.bashrc
root@psql01:/home/ujack# cd ~
root@psql01:~# . .bashrc
root@psql01:~# echo $BACKUP_PATH
/home/backups/
```

Создадим роль в PostgreSQL для выполнения бекапов и дадим ему соответствующие права
```bash
postgres=# create user backup;
CREATE ROLE
postgres=# ALTER ROLE backup NOSUPERUSER;
ALTER ROLE
postgres=# ALTER ROLE backup WITH REPLICATION;
ALTER ROLE
postgres=# GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_start_backup(text, boolean, boolean) TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_stop_backup(boolean, boolean) TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
GRANT
```

Инициализируем наш бэкап
```bash
root@psql01:~# pg_probackup-11 init
INFO: Backup catalog '/home/backups' successfully inited
```

В директории с бэкапами появились каталоги
```bash
root@psql01:~# ll /home/backups
total 0
drwx------ 2 root root 6 Jul 22 13:59 backups
drwx------ 2 root root 6 Jul 22 13:59 wal
```

Инициализируем инстанс postgresql-11
```bash
root@psql01:~# pg_probackup-11 add-instance --instance 'postgresql-11' -D /dbs/pgsql/11/data/
INFO: Instance 'postgresql-11' successfully inited
```

Создадим новую БД
```bash
root@psql01:~# sudo -u postgres psql -c "CREATE DATABASE otus;"
CREATE DATABASE
```

Создадим таблицу с тестовыми данными
```bash
root@psql01:~# sudo -u postgres psql -c "CREATE DATABASE otus;^C
root@psql01:~# sudo -u postgres psql otus -c "create table test(i int);"
CREATE TABLE
root@psql01:~# sudo -u postgres psql otus -c "insert into test values (10), (20), (30);"
INSERT 0 3
root@psql01:~# sudo -u postgres psql otus -c "select * from test;"
 i
----
 10
 20
 30
(3 rows)

```

Посмотрим настройки pg_probackup
```bash
root@psql01:~# pg_probackup-11 show-config --instance postgresql-11
# Backup instance information
pgdata = /dbs/pgsql/11/data
system-identifier = 6820013573612853571
xlog-seg-size = 16777216
# Connection parameters
pgdatabase = root
# Replica parameters
replica-timeout = 5min
# Archive parameters
archive-timeout = 5min
# Logging parameters
log-level-console = INFO
log-level-file = OFF
log-filename = pg_probackup.log
log-rotation-size = 0TB
log-rotation-age = 0d
# Retention parameters
retention-redundancy = 0
retention-window = 0
wal-depth = 0
# Compression parameters
compress-algorithm = none
compress-level = 1
# Remote access parameters
remote-proto = ssh

```

Создадим полный бэкап
```bash
root@psql01:~# pg_probackup-11 backup -d postgres -U backup --instance 'postgresql-11' -b FULL --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.5, instance: postgresql-11, backup ID: RFF635, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
INFO: wait for pg_start_backup()
INFO: Wait for WAL segment /home/backups/backups/postgresql-11/RFF635/database/pg_wal/000000020000000000000037 to be streamed
INFO: PGDATA size: 87MB
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup RFF635
INFO: Backup RFF635 data files are valid
INFO: Backup RFF635 resident size: 103MB
INFO: Backup RFF635 completed
```

Внесем теперь в нашу таблицу test дополнительные данные
```bash
root@psql01:~# sudo -u postgres psql otus -c "insert into test values (4);"
INSERT 0 1
```

Создадим инкрементальную копию
```bash
root@psql01:~# pg_probackup-11 backup --instance 'postgresql-11' -b DELTA --stream --temp-slot -U backup -d postgres
INFO: Backup start, pg_probackup version: 2.5.5, instance: postgresql-11, backup ID: RFF6IZ, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
INFO: wait for pg_start_backup()
INFO: Parent backup: RFF635
INFO: Wait for WAL segment /home/backups/backups/postgresql-11/RFF6IZ/database/pg_wal/000000020000000000000039 to be streamed
INFO: PGDATA size: 87MB
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup RFF6IZ
INFO: Backup RFF6IZ data files are valid
INFO: Backup RFF6IZ resident size: 16MB
INFO: Backup RFF6IZ completed
```

Проверим состояние бэкапов:
```bash
root@psql01:~# pg_probackup-11 show

BACKUP INSTANCE 'postgresql-11'
=========================================================================================================================================
 Instance       Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time   Data   WAL  Zratio  Start LSN   Stop LSN    Status
=========================================================================================================================================
 postgresql-11  11       RFF6IZ  2022-07-22 14:28:13+03  DELTA  STREAM    2/2   10s  175kB  16MB    1.00  0/39000028  0/39000198  OK
 postgresql-11  11       RFF635  2022-07-22 14:18:43+03  FULL   STREAM    2/0   10s   87MB  16MB    1.00  0/37000028  0/37002170  OK

```

Имеем второй кластер
```bash
root@psql01:~# systemctl status postgresql-11-2.service
● postgresql-11-2.service - PostgreSQL 11 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-11-2.service; disabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/postgresql-11-2.service.d
           └─override.conf
   Active: active (running) since Fri 2022-07-22 15:20:50 MSK; 3s ago
     Docs: https://www.postgresql.org/docs/11/static/
  Process: 14231 ExecStartPre=/usr/pgsql-11/bin/postgresql-11-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 14237 (postmaster)
   CGroup: /system.slice/postgresql-11-2.service
           ├─14237 /usr/pgsql-11/bin/postmaster -D /dbs/pgsql/11-2/data/
```

Восстановим резервную копию
```bash
root@psql01:~# systemctl stop postgresql-11-2.service
root@psql01:~# rm -rf /dbs/pgsql/11-2/data/
```

```bash
pg_probackup-11 restore --instance 'postgresql-11' -i 'RFF6IZ' -D /dbs/pgsql/11-2/data/

root@psql01:~# ll /dbs/pgsql/11-2/data/
total 60
-rw-r--r--  1 root root   241 Jul 22 15:25 backup_label
drwx------ 14 root root   162 Jul 22 15:25 base
-rw-------  1 root root    30 Jul 22 15:25 current_logfiles
drwx------  2 root root  4096 Jul 22 15:25 global
drwx------  2 root root     6 Jul 22 15:25 log
drwx------  2 root root     6 Jul 22 15:25 pg_commit_ts
drwx------  2 root root     6 Jul 22 15:25 pg_dynshmem
-rw-------  1 root root  4507 Jul 22 15:25 pg_hba.conf
-rw-------  1 root root  1636 Jul 22 15:25 pg_ident.conf
drwx------  4 root root    68 Jul 22 15:25 pg_logical
drwx------  4 root root    36 Jul 22 15:25 pg_multixact
drwx------  2 root root     6 Jul 22 15:25 pg_notify
drwx------  2 root root     6 Jul 22 15:25 pg_replslot
drwx------  2 root root     6 Jul 22 15:25 pg_serial
drwx------  2 root root     6 Jul 22 15:25 pg_snapshots
drwx------  2 root root     6 Jul 22 15:25 pg_stat
drwx------  2 root root     6 Jul 22 15:25 pg_stat_tmp
drwx------  2 root root     6 Jul 22 15:25 pg_subtrans
drwx------  2 root root     6 Jul 22 15:25 pg_tblspc
drwx------  2 root root     6 Jul 22 15:25 pg_twophase
-rw-------  1 root root     3 Jul 22 15:25 PG_VERSION
drwx------  2 root root    62 Jul 22 15:25 pg_wal
drwx------  2 root root    18 Jul 22 15:25 pg_xact
-rw-------  1 root root   135 Jul 22 15:25 postgresql.auto.conf
-rw-------  1 root root 24120 Jul 22 15:25 postgresql.conf
-rw-r--r--  1 root root   250 Jul 22 15:25 recovery.done
```
Как обычно, работая не из под юзера postgres необходимо менять права на файлы
```bash
root@psql01:~# chown postgres: -R /dbs/pgsql/11-2/data/
root@psql01:~# ll /dbs/pgsql/11-2/data/
total 60
-rw-r--r--  1 postgres postgres   241 Jul 22 15:25 backup_label
drwx------ 14 postgres postgres   162 Jul 22 15:25 base
-rw-------  1 postgres postgres    30 Jul 22 15:25 current_logfiles
drwx------  2 postgres postgres  4096 Jul 22 15:25 global
drwx------  2 postgres postgres     6 Jul 22 15:25 log
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_commit_ts
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_dynshmem
-rw-------  1 postgres postgres  4507 Jul 22 15:25 pg_hba.conf
-rw-------  1 postgres postgres  1636 Jul 22 15:25 pg_ident.conf
drwx------  4 postgres postgres    68 Jul 22 15:25 pg_logical
drwx------  4 postgres postgres    36 Jul 22 15:25 pg_multixact
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_notify
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_replslot
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_serial
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_snapshots
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_stat
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_stat_tmp
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_subtrans
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_tblspc
drwx------  2 postgres postgres     6 Jul 22 15:25 pg_twophase
-rw-------  1 postgres postgres     3 Jul 22 15:25 PG_VERSION
drwx------  2 postgres postgres    62 Jul 22 15:25 pg_wal
drwx------  2 postgres postgres    18 Jul 22 15:25 pg_xact
-rw-------  1 postgres postgres   135 Jul 22 15:25 postgresql.auto.conf
-rw-------  1 postgres postgres 24120 Jul 22 15:25 postgresql.conf
-rw-r--r--  1 postgres postgres   250 Jul 22 15:25 recovery.done
```
Не забыл поменять порт по умолчанию на другой, запускаю кластер:
```bash
root@psql01:~# systemctl start postgresql-11-2.service
root@psql01:~# systemctl status postgresql-11-2.service
● postgresql-11-2.service - PostgreSQL 11 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-11-2.service; disabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/postgresql-11-2.service.d
           └─override.conf
   Active: active (running) since Fri 2022-07-22 15:28:16 MSK; 4s ago
     Docs: https://www.postgresql.org/docs/11/static/
  Process: 14452 ExecStartPre=/usr/pgsql-11/bin/postgresql-11-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 14457 (postmaster)
   CGroup: /system.slice/postgresql-11-2.service
           ├─14457 /usr/pgsql-11/bin/postmaster -D /dbs/pgsql/11-2/data/
```
Проверяем, что мы восстановили с последними внесенными данными перед инкрементальным бэкапом:
```bash
root@psql01:~# su postgres
bash-4.2$ psql -p 5434
could not change directory to "/root": Permission denied
psql (11.15)
Type "help" for help.

postgres=# \l
                                      List of databases
        Name        |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
--------------------+----------+----------+-------------+-------------+-----------------------
 backup_overview    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 backup_overview2   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 data_catalog       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 data_lowlevel      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 log                | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 otus               | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres           | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 replica_switchover | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0          | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                    |          |          |             |             | postgres=CTc/postgres
 template1          | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
                    |          |          |             |             | postgres=CTc/postgres
 testef             | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(11 rows)

postgres=# \c otus
You are now connected to database "otus" as user "postgres".

otus=# select * from test ;
 i
----
 10
 20
 30
  4
(4 rows)
```
