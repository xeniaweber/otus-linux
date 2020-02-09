## Основное задание 
#### Добавление диска в Vagrantfile  
Добавляю еще один диск в файл
```console
:sata5 => {
	:dfile => './sata5.vdi',
	:size => 250, # Megabytes
	:port => 5
},
```
Загружаю Vagrantfile
```console
vagrant up
```
Подключаюсь по SSH
```console
vagrant ssh
```
Проверяю наличие дисков командой lsblk и вижу количество дисков - 5
```console
[vagrant@otuslinux /]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk 
sdc      8:32   0  250M  0 disk 
sdd      8:48   0  250M  0 disk 
sde      8:64   0  250M  0 disk 
```
#### Собираю RAID5
Зануляю суперблоки 
```console
sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e}
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
```
Вывод mdadm: Unrecognised md component device - /dev/sd* говорит о том, что диски ранее не использовались для RAID  
Собираю RAID5
```console
[vagrant@otuslinux /]$ sudo mdadm --create --verbose /dev/md0 -l 5 -n 4 /dev/sd{b,c,d,e}
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
Опция -n указывает количество устройств в RAID, в данном случае 4  
Опция -l указывает номер RAID, в данном случае 5  
Проверяю статус сборки RAID
```console
[vagrant@otuslinux /]$ cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
 ```
 Вывод [UUUU] говорит о количестве юнитов, в данном случае их 4  
 Также проверяю
 ```console
[vagrant@otuslinux /]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Feb  9 14:37:17 2020
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Sun Feb  9 14:37:22 2020
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 9af09035:430e2b7a:1ac13b42:d9f3f710
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde
```
Вывод lsblk
```console
[vagrant@otuslinux /]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda      8:0    0   40G  0 disk  
└─sda1   8:1    0   40G  0 part  /
sdb      8:16   0  250M  0 disk  
└─md0    9:0    0  744M  0 raid5 
sdc      8:32   0  250M  0 disk  
└─md0    9:0    0  744M  0 raid5 
sdd      8:48   0  250M  0 disk  
└─md0    9:0    0  744M  0 raid5 
sde      8:64   0  250M  0 disk  
└─md0    9:0    0  744M  0 raid5 
```
#### Конфигурация для автосборки RAID при загрузке
Создаю директорию mdadm
```console
mkdir /etc/mdadm
```
Создаю файл конфигруации mdadm.conf
```console
[root@otuslinux /]$echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux /]mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```
#### Проверяю работоспособность RAID
"Зафейлю" искусствено одно из блочных устройств
```console
[root@otuslinux /]# mdadm /dev/md0 --fail /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md0
```
Проверяю статус
```console
[root@otuslinux /]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdc[1](F) sdb[0]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [U_UU]
```
Вижу, что на месте одного юнита пробел  
Значит, что второй юнит "зафейлен", а именно /dev/sdc
Удаляю этот диск из RAID
```console
[root@otuslinux /]# mdadm /dev/md0 --remove /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md0
```
Для системы этот диск считается теперь новым. Пробую добавить его в массив
```console
[root@otuslinux /]# mdadm /dev/md0 --add /dev/sdc
mdadm: added /dev/sdc
```
Проверяю
```console
[root@otuslinux /]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Feb  9 14:37:17 2020
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Sun Feb  9 15:38:41 2020
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 9af09035:430e2b7a:1ac13b42:d9f3f710
            Events : 40

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       5       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde
 ```
 Вижу, что диск удачно добавлен  
 #### Создание GPT раздела из 5 партиций
 Создаю раздел GPT на RAID
 ```console
[root@otuslinux \]# parted -s /dev/md0 mklabel gpt
 ```
 Создаю партиции
  ```console
[root@otuslinux \]# parted /dev/md0 mkpart primary ext4 0% 20%
[root@otuslinux \]# parted /dev/md0 mkpart primary ext4 20% 40%
[root@otuslinux \]# parted /dev/md0 mkpart primary ext4 40% 60%
[root@otuslinux \]# parted /dev/md0 mkpart primary ext4 60% 80%
[root@otuslinux \]# parted /dev/md0 mkpart primary ext4 80% 100%
 ```
 Далее создаю на этих партициях файловую систему
 ```console
 for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
 ```
 Монтирую их по каталогам
 ```console
[root@otuslinux \]# mkdir -p /raid/part{1,2,3,4,5}
[root@otuslinux \]# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
```
Получаю следующий вывод lsblk
```console
[vagrant@otuslinux ~]$ lsblk
NAME      MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda         8:0    0    40G  0 disk  
└─sda1      8:1    0    40G  0 part  /
sdb         8:16   0   250M  0 disk  
└─md0       9:0    0   744M  0 raid5 
  ├─md0p1 259:5    0   147M  0 md    /raid/part1
  ├─md0p2 259:6    0 148,5M  0 md    /raid/part2
  ├─md0p3 259:7    0   150M  0 md    /raid/part3
  ├─md0p4 259:8    0 148,5M  0 md    /raid/part4
  └─md0p5 259:9    0   147M  0 md    /raid/part5
sdc         8:32   0   250M  0 disk  
└─md0       9:0    0   744M  0 raid5 
  ├─md0p1 259:5    0   147M  0 md    /raid/part1
  ├─md0p2 259:6    0 148,5M  0 md    /raid/part2
  ├─md0p3 259:7    0   150M  0 md    /raid/part3
  ├─md0p4 259:8    0 148,5M  0 md    /raid/part4
  └─md0p5 259:9    0   147M  0 md    /raid/part5
sdd         8:48   0   250M  0 disk  
└─md0       9:0    0   744M  0 raid5 
  ├─md0p1 259:5    0   147M  0 md    /raid/part1
  ├─md0p2 259:6    0 148,5M  0 md    /raid/part2
  ├─md0p3 259:7    0   150M  0 md    /raid/part3
  ├─md0p4 259:8    0 148,5M  0 md    /raid/part4
  └─md0p5 259:9    0   147M  0 md    /raid/part5
sde         8:64   0   250M  0 disk  
└─md0       9:0    0   744M  0 raid5 
  ├─md0p1 259:5    0   147M  0 md    /raid/part1
  ├─md0p2 259:6    0 148,5M  0 md    /raid/part2
  ├─md0p3 259:7    0   150M  0 md    /raid/part3
  ├─md0p4 259:8    0 148,5M  0 md    /raid/part4
  └─md0p5 259:9    0   147M  0 md    /raid/part5
```
## Задание с * - Vagrantfile, который сразу собирает систему с подключенным рейдом
В Vagrantfile добавляю следующие строки
```console
    config.vm.provision "shell", inline: <<-SHELL
      yum install mdadm -y
      mdadm --zero-superblock --force /dev/sd{b,c,d,e}
      mdadm --create --verbose /dev/md0 -l 5 -n 4 /dev/sd{b,c,d,e}
      mkdir /etc/mdadm
      echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
      mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
      parted -s /dev/md0 mklabel gpt
      parted /dev/md0 mkpart primary ext4 0% 20%
      parted /dev/md0 mkpart primary ext4 20% 40%
      parted /dev/md0 mkpart primary ext4 40% 60%
      parted /dev/md0 mkpart primary ext4 60% 80%
      parted /dev/md0 mkpart primary ext4 80% 100%
      for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
      mkdir -p /raid/part{1,2,3,4,5}
      for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
      SHELL
```
Сам Vagrantfile - [Vagrantfile](https://github.com/xeniaweber/otus-linux/blob/master/hw2/step2/Vagrantfile)
