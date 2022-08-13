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
  






