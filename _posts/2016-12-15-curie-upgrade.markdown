---
layout: post
title:  "Curie upgrade"
date:   2016-12-15 12:00:00
---

Curie - выделенный сервер  арендуемый у Hetzner на протяжении почти 3 лет. Несмотря на внутреннее разделение ресурсов используя виртуализацию на базе VirtualBox, ранее [описанной][1], периодически приходиться сталкиваться с последствиями непродуманного изначального разбиения дисков (используемого по умолчанию в Hetzner):

```
$ df -h
Filesystem            Size  Mounted on
/dev/md2              688G  /
/dev/md1              2.0G   /boot

$ cat /proc/mdstat
Personalities : [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md2 : active raid1 sda3[0] sdb3[1]
md1 : active raid1 sda2[0] sdb2[1]
md0 : active raid1 sda1[0] sdb1[1]
```

Несмотря на надежность софтварного raid1 (зеркало), в такой конфигурации отсутствует возможность:
- нормальной проверки части файловой системы
- использовать разные файловые системы
- невозможность увеличения доступного диского пространства и скорости доступа к диску за счет отказа от использования raid1 для какой-то части дискового пространства.
- Итак, стояла задача переустановки операционной системы вместе с переразбивкой диска на более удачную.

Первый возможный вариант решения - использование [installimage][2] скрипта от Hetzner, который позволяет легко и быстро устанавливать начисто различные дистрибутивы Linux из rescue режима. Однако такой вариант нам неподходил по понятным причинам.

В итоге был выбран второй вариант, по-шагово описанный ниже.

__1.разбиваем зеркало свапа (/dev/md0), освобождаем /dev/sdb1 (4gb), монтируем как /mnt/sdb__

```
#mdadm /dev/md0 -f /dev/sdb1
#mdadm /dev/md0 -r /dev/sdb1
#mke2fs -j /dev/sdb1
```

__2.установка squeeze на /dev/sdb1 при помощи debootstrap__

```
#debootstrap --arch amd64 squeeze /mnt/sdb/ http://ftp.us.debian.org/debian
```

__3.монтируем /dev/, /proc/ и заходим в chroot__

```
#LANG=C chroot /mnt/sdb/ /bin/bash
#mount --bind /dev/ /mnt/sdb/dev/
#mount --bind /proc/ /mnt/sdb/proc/
```

__4.в chroot конфигурируем fstab,network,users,etc__

```
#fstab
#proc            /proc           proc    defaults          0       0
#/dev/sdb1       /               ext3    errors=remount-ro 0       1
#/dev/md0        none            swap    sw                0       0
```

- в mdadm оставляем по умолчанию сканирование всех партиций
- passwd,shadow,gshadow скопировать с мигруемого хоста
- ВАЖНО! в initrd есть свой конфиг для mdadm

__5.добавляем новый пункт в загрузчик__

```
#title           Debian GNU/Linux, kernel 2.6.32-5-amd64
#root            (hd1,0)
#kernel          /boot/vmlinuz-2.6.32-5-amd64 root=/dev/sdb1 ro
#initrd          /boot/initrd.img-2.6.32-5-amd64

#default  2
#fallback 0
```

__6.рестарт__

- монтируем /dev/md1
- aptitude, kernel update
- добавляем в grub/menu.lst запись с новым ядром, ставим на неё default, fallback на предыдущий
- рестарт

__7.дисковый перераздел для /__

/dev/md0 - 4gb - root

```
#swapoff /dev/md0
#mkfs.ext3 /dev/md0
#cfdisk /dev/sda - установить тип фс для /dev/sda1 в 83
#editor /etc/fstab - убрать md0 свап

#mkdir -p /mnt/md0
#mount /dev/md0 /mnt/md0
#cp -dpRx / /mnt/md0
```

- настраиваем загрузку в md0

```
#editor /mnt/md0/etc/fstab - md0 для /
#mkdir -p /mnt/md1
#mount /dev/md1 /mnt/md1
#editor /mnt/md1/grub/menu.lst
```

- добавим в загрузчик

```
#title           Debian GNU/Linux, kernel 2.6.32-5-amd64
#root            (hd0,0)
#kernel          /boot/vmlinuz-??? root=/dev/md0 ro
#initrd          /boot/initrd.img-???
#default 4
#fallback 3
```

- рестарт
- делаем рейд для /

```
#mdadm /dev/md0 --add /dev/sdb1
```

__8.дисковый перераздел для /home__
- выводим /dev/sdb3 из /dev/md2

```
#mdadm /dev/md2 -f /dev/sdb3
#mdadm /dev/md2 -r /dev/sdb3
```

- fdisk - удаляем /dev/sdb3, делаем 2 равных партиции
sdb3 ~ 300gb
sdb4 ~ 300gb

- создаем новый рейд на /dev/sdb3 для /home

```
#mdadm --create --verbose /dev/md3 --level=1 --raid-devices=2 /dev/sdb3 missing
```

- создаем лвм на /dev/sdb4 и /dev/sda4

```
#pvcreate /dev/sda4
#pvcreate /dev/sdb4
#vgcreate vg0 /dev/sda4 /dev/sdb4
```

- создаем стрип лвм /var

```
#lvcreate -i2 -I4 -L20G -nvarlv vg0
#lvcreate -i2 -I4 -L550G -nhomelv vg0
#mkfs.ext3 /dev/vg0/homelv –b 4096 –E stride=1,stripe-width=2
#mkfs.ext3 /dev/vg0/varlv –b 4096 –E stride=1,stripe-width=2
```

- копируем var

```
#mount /dev/vg0/varlv /mnt/varlv
#cp -dpRx /var/ /mnt/varlv
```

[1]: http://blog.kotov.lv/2015/05/planning-the-future-for-curie/

[2]: http://wiki.hetzner.de/index.php/Installimage/ru


