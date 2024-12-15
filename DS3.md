# Начало
## создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в ЯО/Virtual Box/докере
## поставьте на нее PostgreSQL 15 через sudo apt
```bash
apt-get update
apt-get install postgresql-16 postgresql-contrib
```
## проверьте что кластер запущен через sudo -u postgres pg_lsclusters
```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# ps fp $(pgrep post)
    PID TTY      STAT   TIME COMMAND
  16092 ?        Ss     0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
  16093 ?        Ss     0:00  \_ postgres: 16/main: checkpointer
  16094 ?        Ss     0:00  \_ postgres: 16/main: background writer
  16096 ?        Ss     0:00  \_ postgres: 16/main: walwriter
  16097 ?        Ss     0:00  \_ postgres: 16/main: autovacuum launcher
  16098 ?        Ss     0:00  \_ postgres: 16/main: logical replication launcher
```
```bash
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```


## зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
### postgres=# create table test(c1 text);
### postgres=# insert into test values('1');
### \q

```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# su - postgres
postgres@compute-vm-2-2-20-ssd-1734270746062:~$ psql
psql (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
Type "help" for help.

postgres=#  create table test(c1 text);
CREATE TABLE
postgres=# insert into test values ('Hello World');
INSERT 0 1
postgres=# \q
postgres@compute-vm-2-2-20-ssd-1734270746062:~$
```

## остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# systemctl stop postgresql.service
root@compute-vm-2-2-20-ssd-1734270746062:~# ps fp $(pgrep post)
error: list of process IDs must follow p

Usage:
 ps [options]

 Try 'ps --help <simple|list|output|threads|misc|all>'
  or 'ps --help <s|l|o|t|m|a>'
 for additional help text.

For more details see ps(1).
root@compute-vm-2-2-20-ssd-1734270746062:~#
```

### создайте новый диск к ВМ размером 10GB
### добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
### проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего ### будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
### перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
