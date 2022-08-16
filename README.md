# Unit_3_LVM

1. п.1 "уменьшить том под / до 8G" :
- создадим PV, VG- decrease_root и LV new_root для /dev/sdb

vgs
    VG            #PV #LV #SN Attr   VSize   VFree\
  VolGroup00      1   2   0 wz--n- <38.97g    0 \
  decrease_root   1   1   0 wz--n- <10.00g 1.00g
  
lvs 
    LogVol00 VolGroup00    -wi-ao---- <37.47g \                                                   
  LogVol01 VolGroup00    -wi-ao----   1.50g  \                                                  
  new_root decrease_root -wi-a-----  <9.00g 
  
  - создадим файловую систему, примонтируем ее в /mnt
  
  mkfs.xfs /dev/decrease_root/new_root
  mount /dev/decrease_root/new_root /mnt/
  
  sdb                        8:16   0   10G  0 disk \
└─decrease_root-new_root 253:2    0    9G  0 lvm  /mnt

- скопируем данные из / в /mnt

 xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
 
 - Сымитируем текущий root -> сделаем в него chroot и обновим grub:
 
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done\
chroot /mnt/\
grub2-mkconfig -o /boot/grub2/grub.cfg

- обновим образ initrd 

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done

- в файле /boot/grub2/grub.cfg заменим rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root, ребутнемся
- проверим, что root теперь на sdb

sdb                        8:16   0   10G  0 disk 
└─decrease_root-new_root 253:0    0    9G  0 lvm  /

- удаляем LV, создаем его заново с размером 8 Гб
lvremove /dev/VolGroup00/LogVol00\
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

-создаем файловую систему, монтируем, копируем данные

mkfs.xfs /dev/VolGroup00/LogVol00\
mount /dev/VolGroup00/LogVol00 /mnt\
xfsdump -J - /dev/decrease_root/new_root | xfsrestore -J - /mnt

- переконфигурируем grub

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done\
chroot /mnt/\
grub2-mkconfig -o /boot/grub2/grub.cfg\
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done

- ребутимся, проверяем рута, размер

└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  
п.2 Выделяем том под /home

lvcreate -n LV_HOME -l+30%FREE /dev/VolGroup00 #make Logical Volume for /home
mkfs.ext4 /dev/VolGroup00/LV_HOME # make fs
cp -aR /home/* /mnt/ # copy all from home to /mnt
rm -rf /home/ # delete all from home
umount /mnt/
mount /dev/VolGroup00/LV_HOME /home/
echo "`blkid|grep HOME | awk '{print $1}'` /home ext4 defaults 0 0" >> /etc/fstab #insert change to fstab

п.3 выделить том под /var (/var - сделать в mirror)


pvcreate /dev/sd{d,e} # Сreate physical volume
  Physical volume "/dev/sdd" successfully created.\
  Physical volume "/dev/sde" successfully created.


vgcreate VG_VAR /dev/sd{d,e}\ # Create volume group and mirror logical volume
lvcreate -L 900M -m1 -n lv_var VG_VAR:\
lv_var   VG_VAR     rwi-a-r--- 900.00m                                    100.00  

 
mkfs.xfs /dev/VG_VAR/lv_var\ # repeat p.2 actions
cp -aR /var/* /mnt/\
rm -rf /var/*\
umount /mnt/ && mount /dev/VG_VAR/lv_var /var/\
echo "`blkid|grep lv_var:|awk '{print $2}'` /var xfs defaults 0 0" >> /etc/fstab

п. 4 для /home - сделать том для снэпшотов

lvcreate -L 500M -s -n home_snap /dev/vg_home/lv_home

п.5 прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)

Выполнено в п.2 и п.3 с файловыми системами ext4 и xfs

п. 6 

- сгенерировать файлы в /home/
touch file{1..20}

- снять снэпшот
lvcreate -L 500M -s -n home_snap /dev/vg_home/lv_home

- удалить часть файлов

rm -f file{1..20}

- восстановиться со снэпшота
umount /home/ 
lvconvert --merge /dev/vg_home/home_snap

  Merging of volume vg_home/home_snap started.\
  vg_home/lv_home: Merged: 100.00%

mount /home
cd /home
ll

-rw-r--r--. 1 root root     0 Aug 16 15:13 file1\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file10\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file11\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file12\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file13\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file14\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file15\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file16\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file17\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file18\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file19\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file2\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file20\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file3\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file4\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file5\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file6\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file7\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file8\
-rw-r--r--. 1 root root     0 Aug 16 15:13 file9


