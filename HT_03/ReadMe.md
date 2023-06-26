1. Развернул ВМ в Яндекс-облаке.
2. Подключился командой:
ssh -i ~/.ssh/yc_key otus@51.250.99.240
3. Установил постгрес 15 командой:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
4. Проверил, что кластер запущен:
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
5. Зашел в постгрес 
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres psql
could not change directory to "/home/otus": Permission denied
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.
6. Создал таблицу и добавил новую строку:
postgres=# create table test(c1 text);
insert into test values('1');
CREATE TABLE
INSERT 0 1
7. Остановил кластер. Прооверил, что он остановлен:
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
8. Создал в Яндекс-облаке дополнительный диск. Подключил его в ВМ через интерфейс ЯО.
9. Диск появился в списке:
otus@otus-db-pg-vm-1:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.4M  1 loop /snap/core20/1950
loop1    7:1    0  79.9M  1 loop /snap/lxd/22923
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0  53.3M  1 loop /snap/snapd/19361
loop4    7:4    0  53.3M  1 loop /snap/snapd/19457
loop5    7:5    0  63.5M  1 loop /snap/core20/1891
vda    252:0    0     8G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0     8G  0 part /
vdb    252:16   0    10G  0 disk
10. Указал GPT partition:
otus@otus-db-pg-vm-1:~$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.
11. Создал раздел:
otus@otus-db-pg-vm-1:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.
12. Проверяем:
otus@otus-db-pg-vm-1:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.4M  1 loop /snap/core20/1950
loop1    7:1    0  79.9M  1 loop /snap/lxd/22923
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0  53.3M  1 loop /snap/snapd/19361
loop4    7:4    0  53.3M  1 loop /snap/snapd/19457
loop5    7:5    0  63.5M  1 loop /snap/core20/1891
vda    252:0    0     8G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0     8G  0 part /
vdb    252:16   0    10G  0 disk
└─vdb1 252:17   0    10G  0 part
13. Создал файловую систему:
otus@otus-db-pg-vm-1:~$ sudo mkfs.ext4 -L datapartition /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 495a1caf-b381-461d-a934-a6db0b38dc67
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

14. Примонтировал новый диск:
otus@otus-db-pg-vm-1:~$ sudo mkdir -p /mnt/data
otus@otus-db-pg-vm-1:~$ sudo mount -o defaults /dev/vdb1 /mnt/data

15. Проверка:
otus@otus-db-pg-vm-1:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/vda2       7.8G  4.7G  2.8G  63% /
tmpfs           982M     0  982M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           197M  4.0K  197M   1% /run/user/1000
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data

16. Делаю изменения в /etc/fstab:

otus@otus-db-pg-vm-1:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/82aeea96-6d42-49e6-85d5-9071d3c9b6aa / ext4 defaults 0 1

17. Сделал пользователя postgres владельцем /mnt/data:
otus@otus-db-pg-vm-1:~$ sudo chown -R postgres:postgres /mnt/data/
otus@otus-db-pg-vm-1:~$ ls -l /mnt/data
total 16
drwx------ 2 postgres postgres 16384 Jun 26 18:57 lost+found

18. Перенес содержимое /var/lib/postgres/15 в /mnt/data:
otus@otus-db-pg-vm-1:~$ sudo mv /var/lib/postgresql/15 /mnt/data

19. Поменял настройку в postgresql.conf:
otus@otus-db-pg-vm-1:/etc/postgresql/15/main$ sudo nano postgresql.conf

data_directory = '/mnt/data/15/main'            # use data in another directory

20. Запускаю кластер и проверяю:
otus@otus-db-pg-vm-1:/etc/postgresql/15/main$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
otus@otus-db-pg-vm-1:/etc/postgresql/15/main$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log

21. Заходим в постгрес и проверям что таблица на месте:
postgres=# select * from test;
 c1
----
 1
(1 row)
