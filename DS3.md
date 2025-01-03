# Начало
## Физический уровень
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
```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# fdisk -l
Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: CE8F06C4-F24A-4270-979A-DC47B1426910

Device     Start      End  Sectors Size Type
/dev/vda1   2048     4095     2048   1M BIOS boot
/dev/vda2   4096 41943006 41938911  20G Linux filesystem


Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
root@compute-vm-2-2-20-ssd-1734270746062:~#
```
```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# mkdir /mnt/data
root@compute-vm-2-2-20-ssd-1734270746062:~# fdisk /dev/vdb

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0xeab61370.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.
```
```bash

root@compute-vm-2-2-20-ssd-1734270746062:~# mkfs.ext4 /dev/vdb1
mke2fs 1.47.0 (5-Feb-2023)
/dev/vdb1 contains a xfs file system
Proceed anyway? (y,N) y
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: 7ac83d15-a609-4f4f-8718-bf28e4044dc1
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

```
```bash
Создал файл на примонтированном диске, что бы потом проверить его наличие при перезагрузке
 vim /mnt/data/test.tst
root@compute-vm-2-2-20-ssd-1734270746062:~# cat /mnt/data/test.tst
111
```

```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/9ca38502-006d-4f2a-89e1-4c5147e69837 / ext4 defaults 0 1
/dev/disk/by-uuid/7ac83d15-a609-4f4f-8718-bf28e4044dc1  /mnt/data       ext4    defaults 0 0
```
```bash
reboot
root@compute-vm-2-2-20-ssd-1734270746062:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.2M  196M   1% /run
/dev/vda2        20G  3.3G   16G  18% /
tmpfs           984M  1.1M  983M   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
tmpfs           197M   12K  197M   1% /run/user/1000
```
## сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
## перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
```bash
root@compute-vm-2-2-20-ssd-1734270746062:~# systemctl stop postgresql@16-main.service
chown -R postgres:postgres /mnt/data/
mv /var/lib/postgresql/16 /mnt/data
root@compute-vm-2-2-20-ssd-1734270746062:~# pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```
## напишите получилось или нет и почему
### не получилось, потому что я переместил каталог с базой данных в другое место
### т.е. процесс, призапуске, если мы не укажем в pd_ctl -D /путь/до/базы смотрит свой конфигурационный файл postgresql.conf
### секцию FILE LOCATIONS и выбирает каталог БД оттуда. 
### для того что бы база запустилась, надо либо поменять в этой секции местоположение базы, либо при запуске указать путь в опции pg_ctl -D /path_to_database

```bash
vim /etc/postgresql/16/main/postgresql.conf
data_directory = '/mnt/data/16/main'            # use data in another directory
root@compute-vm-2-2-20-ssd-1734270746062:~# pg_ctlcluster 16 main start
root@compute-vm-2-2-20-ssd-1734270746062:~# ps fp $(pgrep post)
    PID TTY      STAT   TIME COMMAND
   1862 ?        Ss     0:00 /usr/lib/postgresql/16/bin/postgres -D /mnt/data/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
   1863 ?        Ss     0:00  \_ postgres: 16/main: checkpointer
   1864 ?        Ss     0:00  \_ postgres: 16/main: background writer
   1866 ?        Ss     0:00  \_ postgres: 16/main: walwriter
   1867 ?        Ss     0:00  \_ postgres: 16/main: autovacuum launcher
   1868 ?        Ss     0:00  \_ postgres: 16/main: logical replication launcher
root@compute-vm-2-2-20-ssd-1734270746062:~#
```
```sql
postgres=# \dt+
                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | test | table | postgres | permanent   | heap          | 16 kB |
(1 row)

postgres=# select * from test ;
     c1
-------------
 Hello World
(1 row)

postgres=#
```
# Данные сохранились
## задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
```bash
root@compute-vm-2-2-20-ssd-1734280619230:~# ps fp $(pgrep post)
    PID TTY      STAT   TIME COMMAND
   4577 ?        Ss     0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
   4579 ?        Ss     0:00  \_ postgres: 16/main: checkpointer
   4580 ?        Ss     0:00  \_ postgres: 16/main: background writer
   4582 ?        Ss     0:00  \_ postgres: 16/main: walwriter
   4583 ?        Ss     0:00  \_ postgres: 16/main: autovacuum launcher
   4584 ?        Ss     0:00  \_ postgres: 16/main: logical replication launcher
root@compute-vm-2-2-20-ssd-1734280619230:~# systemctl stop postgresql@16-main.service
root@compute-vm-2-2-20-ssd-1734280619230:~# rm -rf /var/lib/postgresql/*

```
### я остановил кластер, убедился что он остановлен

```bash
root@compute-vm-2-2-20-ssd-1734280619230:~# ps fp $(pgrep post)
error: list of process IDs must follow p

Usage:
 ps [options]

 Try 'ps --help <simple|list|output|threads|misc|all>'
  or 'ps --help <s|l|o|t|m|a>'
 for additional help text.
```
### к виртуалке 2 примонтировал диск с базой данных от виртуалки1

```bash
For more details see ps(1).
root@compute-vm-2-2-20-ssd-1734280619230:~#
root@compute-vm-2-2-20-ssd-1734280619230:~# mkdir /mnt/data
root@compute-vm-2-2-20-ssd-1734280619230:~# chown postgres:postgres /mnt/data
root@compute-vm-2-2-20-ssd-1734280619230:~# mount /dev/vdb1 /mnt/data/
root@compute-vm-2-2-20-ssd-1734280619230:~# ls -al /mnt/data/
16/         lost+found/
root@compute-vm-2-2-20-ssd-1734280619230:~# ls -al /mnt/data/16/main/
total 88
drwx------ 19 postgres postgres 4096 Dec 15 16:46 .
drwxr-xr-x  3 postgres postgres 4096 Dec 15 14:02 ..
drwx------  5 postgres postgres 4096 Dec 15 14:02 base
drwx------  2 postgres postgres 4096 Dec 15 16:32 global
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_commit_ts
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_dynshmem
drwx------  4 postgres postgres 4096 Dec 15 16:46 pg_logical
drwx------  4 postgres postgres 4096 Dec 15 14:02 pg_multixact
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_notify
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_replslot
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_serial
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_snapshots
drwx------  2 postgres postgres 4096 Dec 15 16:46 pg_stat
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_stat_tmp
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_subtrans
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_tblspc
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_twophase
-rw-------  1 postgres postgres    3 Dec 15 14:02 PG_VERSION
drwx------  3 postgres postgres 4096 Dec 15 14:02 pg_wal
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_xact
-rw-------  1 postgres postgres   88 Dec 15 14:02 postgresql.auto.conf
-rw-------  1 postgres postgres  120 Dec 15 16:31 postmaster.opts
```
### убедился что все примонтировалось
### установил postgresql, убедился что он работает
### затем остановил его
### удалил директорию с базой
```bash
root@compute-vm-2-2-20-ssd-1734280619230:~# apt-get install postgresql-16 postgresql-contrib
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done

root@compute-vm-2-2-20-ssd-1734280619230:~# ps fp $(pgrep post)
    PID TTY      STAT   TIME COMMAND
   4577 ?        Ss     0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgre
   4579 ?        Ss     0:00  \_ postgres: 16/main: checkpointer
   4580 ?        Ss     0:00  \_ postgres: 16/main: background writer
   4582 ?        Ss     0:00  \_ postgres: 16/main: walwriter
   4583 ?        Ss     0:00  \_ postgres: 16/main: autovacuum launcher
   4584 ?        Ss     0:00  \_ postgres: 16/main: logical replication launcher
root@compute-vm-2-2-20-ssd-1734280619230:~# systemctl stop postgresql
postgresql@16-main.service  postgresql.service
root@compute-vm-2-2-20-ssd-1734280619230:~# systemctl stop postgresql@16-main.service
root@compute-vm-2-2-20-ssd-1734280619230:~# ls -al /var/lib/postgresql/16/main/
base/                 pg_logical/           pg_serial/            pg_subtrans/          pg_wal/
global/               pg_multixact/         pg_snapshots/         pg_tblspc/            pg_xact/
pg_commit_ts/         pg_notify/            pg_stat/              pg_twophase/          postgresql.auto.conf
pg_dynshmem/          pg_replslot/          pg_stat_tmp/          PG_VERSION            postmaster.opts
root@compute-vm-2-2-20-ssd-1734280619230:~# rm -rf /var/lib/postgresql/*
```
### База данных не запустилась потому что опять же не нашла директорию
```bash
root@compute-vm-2-2-20-ssd-1734280619230:~# fdisk -l
Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: CE8F06C4-F24A-4270-979A-DC47B1426910

Device     Start      End  Sectors Size Type
/dev/vda1   2048     4095     2048   1M BIOS boot
/dev/vda2   4096 41943006 41938911  20G Linux filesystem


Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0xeab61370

Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 20971519 20969472  10G 83 Linux

root@compute-vm-2-2-20-ssd-1734280619230:~# mkdir /mnt/data
root@compute-vm-2-2-20-ssd-1734280619230:~# chown postgres:postgres /mnt/data
root@compute-vm-2-2-20-ssd-1734280619230:~# mount /dev/vdb1 /mnt/data/
root@compute-vm-2-2-20-ssd-1734280619230:~# ls -al /mnt/data/
16/         lost+found/
root@compute-vm-2-2-20-ssd-1734280619230:~# ls -al /mnt/data/16/main/
total 88
drwx------ 19 postgres postgres 4096 Dec 15 16:46 .
drwxr-xr-x  3 postgres postgres 4096 Dec 15 14:02 ..
drwx------  5 postgres postgres 4096 Dec 15 14:02 base
drwx------  2 postgres postgres 4096 Dec 15 16:32 global
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_commit_ts
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_dynshmem
drwx------  4 postgres postgres 4096 Dec 15 16:46 pg_logical
drwx------  4 postgres postgres 4096 Dec 15 14:02 pg_multixact
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_notify
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_replslot
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_serial
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_snapshots
drwx------  2 postgres postgres 4096 Dec 15 16:46 pg_stat
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_stat_tmp
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_subtrans
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_tblspc
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_twophase
-rw-------  1 postgres postgres    3 Dec 15 14:02 PG_VERSION
drwx------  3 postgres postgres 4096 Dec 15 14:02 pg_wal
drwx------  2 postgres postgres 4096 Dec 15 14:02 pg_xact
-rw-------  1 postgres postgres   88 Dec 15 14:02 postgresql.auto.conf
-rw-------  1 postgres postgres  120 Dec 15 16:31 postmaster.opts
```
### указал в postgresql.conf путь до базы
### Все заработало, данные в таблице остались на месте

```bash
root@compute-vm-2-2-20-ssd-1734280619230:~# vim /etc/postgresql/16/main/postgresql.conf
root@compute-vm-2-2-20-ssd-1734280619230:~# pg_ctlcluster 16 main start
root@compute-vm-2-2-20-ssd-1734280619230:~# su - postgres
postgres@compute-vm-2-2-20-ssd-1734280619230:~$ psql
psql (16.6 (Ubuntu 16.6-0ubuntu0.24.04.1))
Type "help" for help.

postgres=# select * from test;
     c1
-------------
 Hello World
(1 row)

postgres=#

```

