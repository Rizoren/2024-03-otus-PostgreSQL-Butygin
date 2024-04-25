> Занятие 6  
Установка и настройка PostgreSQL.
---
Конфигурирование и подключение внешнего диска
---
Задаем конфигурацю wsl.conf для автоматического монтирования и прочего:
```bash
root@MSI:/# nano etc/wsl.conf
[boot]
systemd=true

[automount]
enabled = true
root = /mnt
options = "metadata,umask=22,fmask=11"
mountFsTab = true

[network]
generateHosts = true
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = true
```
1 сценарий подключения виртуального диска:
```ps
PS > wsl -l -v
  NAME                   STATE           VERSION
* Ubuntu                 Running         2
  docker-desktop-data    Running         2
  docker-desktop         Running         2
  Debian                 Running         2

PS > GET-CimInstance -query "SELECT * from Win32_DiskDrive"

DeviceID           Caption                        Partitions Size          Model
--------           -------                        ---------- ----          -----
\\.\PHYSICALDRIVE2 Виртуальный диск (Майкрософт)  1          10733990400   Виртуальный диск (Майкрософт)

PS > wsl -d Ubuntu --shutdown
PS > wsl --mount \\.\PHYSICALDRIVE2 --type ntfs
Операция успешно завершена.
PS > wsl -d Ubuntu
```

```bash
root@MSI:/# mkdir otus
root@MSI:/# sudo blkid  /dev/sdd2
/dev/sdd2: LABEL="Otus" BLOCK_SIZE="512" UUID="523C80493C802A55" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="860bc003-28ce-495c-acb6-1e71799f3893"

root@MSI:/# nano etc/fstab
# <file system> <dir> <type> <options> <dump> <pass>
PARTUUID=860bc003-28ce-495c-acb6-1e71799f3893 /otus ntfs defaults 0 2

root@MSI:/# mount -t ntfs /dev/sdd2 /otus
# Для Debian монтирование ntfs через ntfs-3g /dev/sdd2 /otus

root@MSI:/# lsblk -l /dev/sdd
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sdd    8:48   0  10G  0 disk
sdd1   8:49   0  16M  0 part
sdd2   8:50   0  10G  0 part /otus
```
2 сценарий, актуален если диск уже размечен и подключен в хост-машине под Windows (воспользовался им)
```bash
root@MSI:/# sudo mount -t drvfs O: /mnt/o
```
---
Перенос кластера на подключенный диск
---
***Перед переносом необходимо остановить кластер!***

Перенос данных кластера лучше осуществлять через rsync (владелец и разрешения будут скопированы):
```bash
root@MSI:/# sudo rsync -av /var/lib/postgresql /mnt/o/data
```
Если это делать через mv, то придется установить владельца и разрешения:
```bash
root@MSI:/# mkdir /mnt/o/data
root@MSI:/# mv /var/lib/postgresql /mnt/o/data
root@MSI:/# chown -R postgres:postgres /mnt/o/data/
root@MSI:/# chmod -R 700 /mnt/o/data/postgresql/16/main
```
---
Запуск кластера
---
Немедленный запуск кластера приведет к ошибке, т.к. по факту его не будет там где ождает СУБД.

Для этого необходимо изменить значение **data_directory** (PGDATA) в **postgresql.conf** и запустить кластер
```bash
root@MSI:/# nano /etc/postgresql/16/main/postgresql.conf

...
# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.

data_directory = '/mnt/o/data/postgresql/16/main'       # '/var/lib/postgresql/16/main'         # use data in another directory
...

root@MSI:/# pg_ctlcluster 16 main restart
```

---
*Запуск кластера c другого инстанса ВМ
---
Если все сделать корректно, то все работает. Новый инстанс удаляет старый PID-файл и все запускается. 
```bash
root@MSI:/# pg_ctlcluster 16 main restart
Removed stale pid file.
```
А так могут возникать проблемы с правами, их может понадобится перевыдать на каталог и файлы кластера.

Не получилось подцепиться из PG установленного локально на Windows используя кластер на виртуальном диске (разные локали/кодировки)
