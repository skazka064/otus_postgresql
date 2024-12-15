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
