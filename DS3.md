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
root@compute-vm-2-2-20-ssd-1734270746062:~# pg_ctlcluster 16 main stop

root@compute-vm-2-2-20-ssd-1734280619230:~# ps fp $(pgrep post)
error: list of process IDs must follow p

Usage:
 ps [options]

 Try 'ps --help <simple|list|output|threads|misc|all>'
  or 'ps --help <s|l|o|t|m|a>'
 for additional help text.

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
root@compute-vm-2-2-20-ssd-1734280619230:~#
     ┌────────────────────────────────────────────────────────────────────┐
     │                • MobaXterm Personal Edition v22.1 •                │
     │              (X server, SSH client and network tools)              │
     │                                                                    │
     │ ⮞ Your computer drives are accessible through the /drives path     │
     │ ⮞ Your DISPLAY is set to 192.168.1.139:0.0                         │
     │ ⮞ When using SSH, your remote DISPLAY is automatically forwarded   │
     │ ⮞ Each command status is specified by a special symbol (✓ or ✗)    │
     │                                                                    │
     │ • Important:                                                       │
     │ This is MobaXterm Personal Edition. The Professional edition       │
     │ allows you to customize MobaXterm for your company: you can add    │
     │ your own logo, your parameters, your welcome message and generate  │
     │ either an MSI installation package or a portable executable.       │
     │ We can also modify MobaXterm or develop the plugins you need.      │
     │ For more information: https://mobaxterm.mobatek.net/download.html  │
     └────────────────────────────────────────────────────────────────────┘

[2024-12-15 19:41.30]  ~
[admin.DESKTOP-H5GAVV0] ⮞ ssh -l user 51.250.32.73
Warning: Permanently added '51.250.32.73' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-49-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Dec 15 04:41:35 PM UTC 2024

  System load:  0.18               Processes:             146
  Usage of /:   14.8% of 19.59GB   Users logged in:       0
  Memory usage: 10%                IPv4 address for eth0: 10.130.0.33
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

40 updates can be applied immediately.
20 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

/usr/bin/xauth:  file /home/user/.Xauthority does not exist
user@compute-vm-2-2-20-ssd-1734280619230:~$ sudo -i
root@compute-vm-2-2-20-ssd-1734280619230:~# apt-get update
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Hit:2 http://archive.ubuntu.com/ubuntu noble-updates InRelease
Hit:3 http://archive.ubuntu.com/ubuntu noble-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu noble-security InRelease
Reading package lists... Done
root@compute-vm-2-2-20-ssd-1734280619230:~# apt-get install postgresql-16 postgresql-contrib
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libcommon-sense-perl libjson-perl libjson-xs-perl libllvm17t64 libpq5 libtypes-serialiser-perl postgresql-client-16 postgresql-client-common
  postgresql-common ssl-cert
Suggested packages:
  postgresql-doc-16
The following NEW packages will be installed:
  libcommon-sense-perl libjson-perl libjson-xs-perl libllvm17t64 libpq5 libtypes-serialiser-perl postgresql-16 postgresql-client-16
  postgresql-client-common postgresql-common postgresql-contrib ssl-cert
0 upgraded, 12 newly installed, 0 to remove and 36 not upgraded.
Need to get 43.5 MB of archives.
After this operation, 175 MB of additional disk space will be used.
Do you want to continue? [Y/n] Y
Get:1 http://archive.ubuntu.com/ubuntu noble/main amd64 libjson-perl all 4.10000-1 [81.9 kB]
Get:2 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-client-common all 257build1.1 [36.4 kB]
Get:3 http://archive.ubuntu.com/ubuntu noble/main amd64 ssl-cert all 1.1.2ubuntu1 [17.8 kB]
Get:4 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-common all 257build1.1 [161 kB]
Get:5 http://archive.ubuntu.com/ubuntu noble/main amd64 libcommon-sense-perl amd64 3.75-3build3 [20.4 kB]
Get:6 http://archive.ubuntu.com/ubuntu noble/main amd64 libtypes-serialiser-perl all 1.01-1 [11.6 kB]
Get:7 http://archive.ubuntu.com/ubuntu noble/main amd64 libjson-xs-perl amd64 4.030-2build3 [83.6 kB]
Get:8 http://archive.ubuntu.com/ubuntu noble/main amd64 libllvm17t64 amd64 1:17.0.6-9ubuntu1 [26.2 MB]
Get:9 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 libpq5 amd64 16.6-0ubuntu0.24.04.1 [141 kB]
Get:10 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-client-16 amd64 16.6-0ubuntu0.24.04.1 [1,271 kB]
Get:11 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-16 amd64 16.6-0ubuntu0.24.04.1 [15.5 MB]
Get:12 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 postgresql-contrib all 16+257build1.1 [11.6 kB]
Fetched 43.5 MB in 3s (16.7 MB/s)
Preconfiguring packages ...
Selecting previously unselected package libjson-perl.
(Reading database ... 124361 files and directories currently installed.)
Preparing to unpack .../00-libjson-perl_4.10000-1_all.deb ...
Unpacking libjson-perl (4.10000-1) ...
Selecting previously unselected package postgresql-client-common.
Preparing to unpack .../01-postgresql-client-common_257build1.1_all.deb ...
Unpacking postgresql-client-common (257build1.1) ...
Selecting previously unselected package ssl-cert.
Preparing to unpack .../02-ssl-cert_1.1.2ubuntu1_all.deb ...
Unpacking ssl-cert (1.1.2ubuntu1) ...
Selecting previously unselected package postgresql-common.
Preparing to unpack .../03-postgresql-common_257build1.1_all.deb ...
Adding 'diversion of /usr/bin/pg_config to /usr/bin/pg_config.libpq-dev by postgresql-common'
Unpacking postgresql-common (257build1.1) ...
Selecting previously unselected package libcommon-sense-perl:amd64.
Preparing to unpack .../04-libcommon-sense-perl_3.75-3build3_amd64.deb ...
Unpacking libcommon-sense-perl:amd64 (3.75-3build3) ...
Selecting previously unselected package libtypes-serialiser-perl.
Preparing to unpack .../05-libtypes-serialiser-perl_1.01-1_all.deb ...
Unpacking libtypes-serialiser-perl (1.01-1) ...
Selecting previously unselected package libjson-xs-perl.
Preparing to unpack .../06-libjson-xs-perl_4.030-2build3_amd64.deb ...
Unpacking libjson-xs-perl (4.030-2build3) ...
Selecting previously unselected package libllvm17t64:amd64.
Preparing to unpack .../07-libllvm17t64_1%3a17.0.6-9ubuntu1_amd64.deb ...
Unpacking libllvm17t64:amd64 (1:17.0.6-9ubuntu1) ...
Selecting previously unselected package libpq5:amd64.
Preparing to unpack .../08-libpq5_16.6-0ubuntu0.24.04.1_amd64.deb ...
Unpacking libpq5:amd64 (16.6-0ubuntu0.24.04.1) ...
Selecting previously unselected package postgresql-client-16.
Preparing to unpack .../09-postgresql-client-16_16.6-0ubuntu0.24.04.1_amd64.deb ...
Unpacking postgresql-client-16 (16.6-0ubuntu0.24.04.1) ...
Selecting previously unselected package postgresql-16.
Preparing to unpack .../10-postgresql-16_16.6-0ubuntu0.24.04.1_amd64.deb ...
Unpacking postgresql-16 (16.6-0ubuntu0.24.04.1) ...
Selecting previously unselected package postgresql-contrib.
Preparing to unpack .../11-postgresql-contrib_16+257build1.1_all.deb ...
Unpacking postgresql-contrib (16+257build1.1) ...
Setting up postgresql-client-common (257build1.1) ...
Setting up libpq5:amd64 (16.6-0ubuntu0.24.04.1) ...
Setting up libcommon-sense-perl:amd64 (3.75-3build3) ...
Setting up libllvm17t64:amd64 (1:17.0.6-9ubuntu1) ...
Setting up ssl-cert (1.1.2ubuntu1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/ssl-cert.service → /usr/lib/systemd/system/ssl-cert.service.
Setting up libtypes-serialiser-perl (1.01-1) ...
Setting up libjson-perl (4.10000-1) ...
Setting up libjson-xs-perl (4.030-2build3) ...
Setting up postgresql-client-16 (16.6-0ubuntu0.24.04.1) ...
update-alternatives: using /usr/share/postgresql/16/man/man1/psql.1.gz to provide /usr/share/man/man1/psql.1.gz (psql.1.gz) in auto mode
Setting up postgresql-common (257build1.1) ...

Creating config file /etc/postgresql-common/createcluster.conf with new version
Building PostgreSQL dictionaries from installed myspell/hunspell packages...
Removing obsolete dictionary files:
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql.service → /usr/lib/systemd/system/postgresql.service.
Setting up postgresql-16 (16.6-0ubuntu0.24.04.1) ...
Creating new PostgreSQL cluster 16/main ...
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/main --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/16/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Setting up postgresql-contrib (16+257build1.1) ...
Processing triggers for man-db (2.12.0-4build2) ...
Processing triggers for libc-bin (2.39-0ubuntu8.3) ...
Scanning processes...
Scanning linux images...

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
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
root@compute-vm-2-2-20-ssd-1734280619230:~# rm -rf /var/lib/postgresql/16/main/
base/                 pg_logical/           pg_serial/            pg_subtrans/          pg_wal/
global/               pg_multixact/         pg_snapshots/         pg_tblspc/            pg_xact/
pg_commit_ts/         pg_notify/            pg_stat/              pg_twophase/          postgresql.auto.conf
pg_dynshmem/          pg_replslot/          pg_stat_tmp/          PG_VERSION            postmaster.opts
root@compute-vm-2-2-20-ssd-1734280619230:~# rm -rf /var/lib/postgresql/*
root@compute-vm-2-2-20-ssd-1734280619230:~# Connection to 51.250.32.73 closed by remote host.
Connection to 51.250.32.73 closed.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[2024-12-15 19:47.48]  ~
[admin.DESKTOP-H5GAVV0] ⮞ ssh -l user 51.250.32.73
Permission denied (publickey).
                                                                                                                                             ✗
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[2024-12-15 19:49.09]  ~
[admin.DESKTOP-H5GAVV0] ⮞ ssh -l user 51.250.32.73
Warning: Permanently added '51.250.32.73' (ECDSA) to the list of known hosts.
Permission denied (publickey).
                                                                                                                                             ✗
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[2024-12-15 19:49.14]  ~
[admin.DESKTOP-H5GAVV0] ⮞ ssh -l user 51.250.32.73
Permission denied (publickey).
                                                                                                                                             ✗
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[2024-12-15 19:50.41]  ~
[admin.DESKTOP-H5GAVV0] ⮞ ssh -l user 130.193.46.26
Warning: Permanently added '130.193.46.26' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-49-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Dec 15 04:51:13 PM UTC 2024

  System load:  0.85               Processes:             142
  Usage of /:   16.0% of 19.59GB   Users logged in:       0
  Memory usage: 10%                IPv4 address for eth0: 10.130.0.33
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

40 updates can be applied immediately.
20 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Sun Dec 15 16:41:37 2024 from 62.217.190.117
user@compute-vm-2-2-20-ssd-1734280619230:~$ sudo -i
root@compute-vm-2-2-20-ssd-1734280619230:~# ps fp $(pgrep post)
error: list of process IDs must follow p

Usage:
 ps [options]

 Try 'ps --help <simple|list|output|threads|misc|all>'
  or 'ps --help <s|l|o|t|m|a>'
 for additional help text.

For more details see ps(1).
root@compute-vm-2-2-20-ssd-1734280619230:~#
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
