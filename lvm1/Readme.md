# Домашнее задание LVM-1#
На виртуальной машине centos/7 -v. 1804.2

1) Уменьшить том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделать в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор)

Работа со снапшотами:
- сгенерить файлы в /home/
- снять снапшот
- удалить часть файлов
- восстановиться со снапшота


В файле Vagrant прописана предварительная установка пакетов: 

        yum install -y mdadm smartmontools hdparm gdisk xfsdump
       
Поднимаем виртуальную машину, подключаемся к ней, переключаемся в sudo:  

    $ vagrant up
    ...
    lvm: 
    lvm: Complete!

    $ vagrant ssh
    [vagrant@lvm ~]$ sudo -i 
    [root@lvm ~]#  
    
Уменьшаем том / до 8G
======================

Создаём, размечаем и монтируем том для / раздела:  

    [root@lvm ~]# pvcreate /dev/sdb
      Physical volume "/dev/sdb" successfully created. 
    
    [root@lvm ~]# vgcreate vg_root /dev/sdb
      Volume group "vg_root" successfully created
      
    [root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
      Logical volume "lv_root" created.
     
    [root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
      meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
               =                       sectsz=512   attr=2, projid32bit=1
               =                       crc=1        finobt=0, sparse=0
      data     =                       bsize=4096   blocks=2620416, imaxpct=25
               =                       sunit=0      swidth=0 blks
      naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
      log      =internal log           bsize=4096   blocks=2560, version=2
               =                       sectsz=512   sunit=0 blks, lazy-count=1
      realtime =none                   extsz=4096   blocks=0, rtextents=0 
      
    [root@lvm ~]# mount /dev/vg_root/lv_root /mnt 
    
С помощью xfsdump/xfsrestore копируем данные с / в /mnt: 
    
    [root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
      ...
      xfsdump: dump complete: 15 seconds elapsed
      xfsdump: Dump Status: SUCCESS
      xfsrestore: restore complete: 15 seconds elapsed
      xfsrestore: Restore Status: SUCCESS
      
Убеждаемся, что команда сработала: 
        
        [root@lvm ~]# ll /mnt
            total 12
            lrwxrwxrwx.  1 root    root       7 Dec 21 16:27 bin -> usr/bin
            drwxr-xr-x.  2 root    root       6 May 12  2018 boot
            drwxr-xr-x.  2 root    root       6 May 12  2018 dev
            drwxr-xr-x. 79 root    root    8192 Dec 21 15:53 etc
            drwxr-xr-x.  3 root    root      21 May 12  2018 home
            lrwxrwxrwx.  1 root    root       7 Dec 21 16:27 lib -> usr/lib
            lrwxrwxrwx.  1 root    root       9 Dec 21 16:27 lib64 -> usr/lib64
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 media
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 mnt
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 opt
            drwxr-xr-x.  2 root    root       6 May 12  2018 proc
            dr-xr-x---.  3 root    root     149 Dec 21 15:52 root
            drwxr-xr-x.  2 root    root       6 May 12  2018 run
            lrwxrwxrwx.  1 root    root       8 Dec 21 16:27 sbin -> usr/sbin
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 srv
            drwxr-xr-x.  2 root    root       6 May 12  2018 sys
            drwxrwxrwt.  8 root    root     193 Dec 21 16:21 tmp
            drwxr-xr-x. 13 root    root     155 May 12  2018 usr
            drwxrwxr-x.  2 vagrant vagrant   25 Dec 21 15:36 vagrant
            drwxr-xr-x. 18 root    root     254 Dec 21 15:51 var
            
Вносим изменения в grub, чтобы при старте подхватывался новый раздел / :

    [root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
    [root@lvm ~]# chroot /mnt/
    [root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
      Generating grub configuration file ...
      Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
      Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
      done
      
Обновляем образ initrd:

    [root@lvm boot]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
    s/.img//g"` --force; done 
        *** Store current command line parameters ***
        *** Creating image file ***
        *** Creating image file done ***
        *** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

Изменяем запись в /boot/grub2/grub.cfg на новый раздел root (вместо **rd.lvm.lv=VolGroup00/LogVol00** вставляем **rd.lvm.lv=vg_root/lv_root**)
    
    [root@lvm ~]# sed -i 's|rd.lvm.lv=VolGroup00/LogVol00|rd.lvm.lv=vg_root/lv_root|' /boot/grub2/grub.cfg 

Перезагружаем виртуальную машину и проверяем, какие разделы смонтировались

    [root@lvm ~]# lsblk
        NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
        sda                       8:0    0   40G  0 disk 
        ├─sda1                    8:1    0    1M  0 part 
        ├─sda2                    8:2    0    1G  0 part /boot
        └─sda3                    8:3    0   39G  0 part 
          ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
          └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm  
        sdb                       8:16   0   10G  0 disk 
        └─vg_root-lv_root       253:0    0   10G  0 lvm  /
        sdc                       8:32   0    2G  0 disk 
        sdd                       8:48   0    1G  0 disk 
        sde                       8:64   0    1G  0 disk 

Удаляем старый LV, вместо него создаём новый на 8G 
    
    [root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
        Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
        Logical volume "LogVol00" successfully removed

    [root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
        WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
        Wiping xfs signature on /dev/VolGroup00/LogVol00.
        Logical volume "LogVol00" created.

Размечаем, монтируем новый том, копируем на него данные:
        
    [root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
        meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
                 =                       sectsz=512   attr=2, projid32bit=1
                 =                       crc=1        finobt=0, sparse=0
        data     =                       bsize=4096   blocks=2097152, imaxpct=25
                 =                       sunit=0      swidth=0 blks
        naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
        log      =internal log           bsize=4096   blocks=2560, version=2
                 =                       sectsz=512   sunit=0 blks, lazy-count=1
        realtime =none                   extsz=4096   blocks=0, rtextents=0
    
    [root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt
    [root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
        xfsdump: dump complete: 13 seconds elapsed
        xfsdump: Dump Status: SUCCESS
        xfsrestore: restore complete: 14 seconds elapsed
        xfsrestore: Restore Status: SUCCESS

Заново настраиваем grub:

    [root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
    [root@lvm ~]# chroot /mnt/
    [root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
        Generating grub configuration file ...
        Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
        Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
        done
        
    [root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
        > s/.img//g"` --force; done   
    
Выделяем том под /var, зеркалируем его:

    [root@lvm boot]# pvcreate /dev/sdc /dev/sdd
          Physical volume "/dev/sdc" successfully created.
          Physical volume "/dev/sdd" successfully created.
    
    [root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
          Volume group "vg_var" successfully created

    [root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
          Rounding up size to full physical extent 952.00 MiB
          Logical volume "lv_var" created.

Создаём ФС и копируем на новый раздел содержимое /var    
    
    [root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var
        mke2fs 1.42.9 (28-Dec-2013)
        Filesystem label=
        OS type: Linux
        Block size=4096 (log=2)
        Fragment size=4096 (log=2)
        Stride=0 blocks, Stripe width=0 blocks
        60928 inodes, 243712 blocks
        12185 blocks (5.00%) reserved for the super user
        First data block=0
        Maximum filesystem blocks=249561088
        8 block groups
        32768 blocks per group, 32768 fragments per group
        7616 inodes per group
        Superblock backups stored on blocks: 
                32768, 98304, 163840, 229376

        Allocating group tables: done                            
        Writing inode tables: done                            
        Creating journal (4096 blocks): done
        Writing superblocks and filesystem accounting information: done


    [root@lvm boot]# mount /dev/vg_var/lv_var /mnt/
    [root@lvm boot]# rsync -avHPSAX /var/ /mnt/
        sending incremental file list
        ./
        .updated
                    163 100%    0.00kB/s    0:00:00 (xfr#1, ir-chk=1026/1028)

        sent 130,352 bytes  received 575 bytes  261,854.00 bytes/sec
        total size is 218,803,538  speedup is 1,671.19

Отмонтируем старый /var и монтируем новый 

    [root@lvm boot]# umount /mnt
    [root@lvm boot]# mount /dev/vg_var/lv_var /var
    
Вносим изменения в fstab для автоматического монтирования нового раздела

    [root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
    
Перезагружаемся, смотрим, что натворили:

    [root@lvm ~]# lsblk
        NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
        sda                        8:0    0   40G  0 disk 
        ├─sda1                     8:1    0    1M  0 part 
        ├─sda2                     8:2    0    1G  0 part /boot
        └─sda3                     8:3    0   39G  0 part 
          ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
          └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
        sdb                        8:16   0   10G  0 disk 
        └─vg_root-lv_root        253:2    0   10G  0 lvm  
        sdc                        8:32   0    2G  0 disk 
        ├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
        │ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
        └─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm  
          └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
        sdd                        8:48   0    1G  0 disk 
        ├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
        │ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
        └─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm  
          └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
        sde                        8:64   0    1G  0 disk 
        
Удаляем временные разделы

    [root@lvm ~]# lvremove /dev/vg_root/lv_root 
        Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
          Logical volume "lv_root" successfully removed

    [root@lvm ~]# vgremove /dev/vg_root
        Volume group "vg_root" successfully removed
        
    [root@lvm ~]# pvremove /dev/sdb
        Labels on physical volume "/dev/sdb" successfully wiped.

По той же схеме выделяем том под /home
        
    [root@lvm ~]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
          Logical volume "LogVol_Home" created.
    
    [root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home 
        meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
                 =                       sectsz=512   attr=2, projid32bit=1
                 =                       crc=1        finobt=0, sparse=0
        data     =                       bsize=4096   blocks=524288, imaxpct=25
                 =                       sunit=0      swidth=0 blks
        naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
        log      =internal log           bsize=4096   blocks=2560, version=2
                 =                       sectsz=512   sunit=0 blks, lazy-count=1
        realtime =none                   extsz=4096   blocks=0, rtextents=0
   
         

    [root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/
    [root@lvm ~]# cp -aR /home/* /mnt/
    [root@lvm ~]# rm -rf /home/*
    [root@lvm ~]# umount /mnt
    [root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/
    [root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
    
Снимаем снапшоты с /home
========================
    
Создадим группу файлов 
        
    [root@lvm ~]# touch /home/file{1..20}
    
Создадим снапшот

    [root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
          Rounding up size to full physical extent 128.00 MiB
          Logical volume "home_snap" created.
          
Затем удалим часть файлов

    [root@lvm ~]# rm -f /home/file{1..10}
    [root@lvm ~]# ll /home
        total 0
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file11
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file12
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file13
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file14
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file15
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file16
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file17
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file18
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file19
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file20
        drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant

    
И восстановим их со снапшота

     [root@lvm ~]# umount /home
     [root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap 
          Merging of volume VolGroup00/home_snap started.
          VolGroup00/LogVol_Home: Merged: 100.00%
     [root@lvm ~]# mount /home
     [root@lvm ~]# ll /home/
        total 0
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file1
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file10
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file11
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file12
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file13
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file14
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file15
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file16
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file17
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file18
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file19
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file2
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file20
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file3
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file4
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file5
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file6
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file7
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file8
        -rw-r--r--. 1 root    root     0 Dec 22 13:04 file9
        drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant

Таким образом, убеждаемся, что задача по восстановлению данных выполнена.

     
  
     
    

    
    


