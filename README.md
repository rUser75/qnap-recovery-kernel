# qnap-recovery-kernel
a minimal QTS kernel running under qemu with lvm support

## Indrodution 
some time ago I've helped a friend of my to recover data from him  QNAP that it wouldn't turn on anymore.

the data volume was on a LVM volume build on top of a md raid, this is the standard for QNAP devices ( it is at least for the two-disc ones ). 

my first attempt was to connect one of the hdd on a linux computer to mount the volume but I was surprised to find out that I'm not able to do this. Why?
QNAP doesn't use a standard Linux LVM but a custom compile that it is a little bit different and that's why the data recovery software didn't work.

after a lot of time we have turned ON the old QNAP with another hdd after that we were able to recover the data on the old hdd.
but... if the QNAP is definely broken?
for this reason I decided to assemble this kernel starting at a qnap x86 device firmware

## before start
if the NAS have two disks before start it is important to mark the disks to remember the correct position.
for me it is also important don't work directly on the disks but on a copy. 
the copy of the data partition can be do using some tool as dd or ddrescue.
> ** If you continue the read I suppose that do you know how to use a linux and that there ara no warranty that all works as expected...**

you need a linux box 

## make copy of the data partition
attach one of the disks into a linux system (is it possible to use sata to usb adapter, a usb box, directly attached ...)
using fdisk to determine to data partition, it is the big one for example (usually the partiotion 3)
```
# fdisk -l /dev/sda

Disk /dev/sda: 2000.3 GB, 2000398934016 bytes
255 heads, 63 sectors/track, 243201 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1               1          66      530125   83  Linux
/dev/sda2              67         132      530142   83  Linux
/dev/sda3             133      243138  1951945693   83  Linux
/dev/sda4          243139      243200      498012   83  Linux
```
after identified the right partion, using another hdd for save the image file you can use dd or ddrescue

```
dd if=/dev/sda3 of=/otherstoragepath/sda3.img bs=8k 
```
or 
```
ddrescue -n /dev/sda3 /otherstoragepath/sda3.img /pathtologfile/logfile
```
> **note** using ddrescue you can try to fix some I/O errors if the source hdd was damaged and specifing the logfile (now called domain-mapfile) you can stop end resume the copy
```
dnf -y install ddrescue (on centos)
```
or
```
add-apt-repository universe
apt-get install gddrescue  (for ubuntu)
```

## try to recover.
you need a linux box (centos, ubuntu live are tested) with sufficient free space for copy the recovered data (with an external usb disk for example)
install the qemu packages using 
```
dnf install -y qemu-kvm (for centos)

apt-get  install qemu-kvm (for ubuntu)
```
download the kernel image from the menu release (https://github.com/rUser75/qnap-recovery-kernel/releases/tag/first)
run the follow command to run the custom kernel in a virtual machine on redhat (on ubuntu repalce /usr/libexec/qemu-kvm with qemu-system-x86_64)

```
/usr/libexec/qemu-kvm \
-kernel bzImage \
-initrd initrdC.boot \
-append "root=/dev/ram0 rw init=/bin/busybox console=ttyS0" \
-nographic -m 1024M \
-serial mon:stdio \
-drive file=sdb3.img,format=raw \
-drive file=/dev/disk_were_copy,format=raw \
```
> sdb3.img is the name of the image file or if you want use the disk the block device of the disk (/dev/sdXXXX)

> /dev/disk_were_copy is the external disk where you can copy data

**note:**  inside the VM the first row -drive become /dev/sda the second /dev/sdb etc

if all works as expected the VM starts.
```
Welcome to the recovery kernel for QNAP product

use admin/admin for login

NAS login:
```

now we can try to mount open the volume ( you can use the ddrescue_volume.img for the test)

```
[~] # mdadm --examine --scan
ARRAY /dev/md/1  metadata=1.0 UUID=e767caf1:30504500:f7da4b48:92bb490e name=1

[~] # mdadm --assemble --run -o /dev/md1 --uuid=e767caf1:30504500:f7da4b48:92b>
[  185.188681] md: md1 stopped.
[  185.197586] md/raid1:md1: active with 1 out of 2 mirrors
[  185.199758] md1: detected capacity change from 0 to 32833536
mdadm: /dev/md1 has been started with 1 drive (out of 2).


[~] # vgs
  VG   #PV #LV #SN Attr   VSize  VFree
  vg1    1   3   1 wz--n- 28.00m    0

[~] # lvs
  LV        VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1       vg1  owi---tz-- 12.00m tp1
  snapShots vg1  swi---s---  4.00m      lv1
  tp1       vg1  twi---tz-- 20.00m

we need to active the volume lv1 (or use the proper volume name if it was different)

[~] # lvchange -ay vg1/lv1
[  312.500140] device-mapper: thin metadata: __create_persistent_data_objects: block manger get correctly
[  312.556542] device-mapper: thin metadata: dm_pool_set_mapped_threshold: threshold: 0 total_mapped_blocks: 192 new_threshold: 320
[  312.561102] device-mapper: thin: maybe_resize_data_dev: expand pool origin max threshold to 320
[  312.569190] device-mapper: snapshots: Snapshot is marked invalid.


[~] # lvs
  LV        VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1       vg1  owi-a-tz-- 12.00m tp1         100.00
  snapShots vg1  swi-I-s---  4.00m      lv1    100.00
  tp1       vg1  twi---tz-- 20.00m             60.00  36.91

now the volume lv1 is active and now we can try to mount it


[~] # mount -t ext4 -o ro /dev/vg1/lv1 /mnt
[  409.386651] ext4_init_reserve_inode_table0: dm-4, 2
[  409.388596] ext4_init_reserve_inode_table2: dm-4, 2, 0, 0, 1024
[  409.390997] EXT4-fs (dm-4): mounted filesystem (<none>) with ordered data mode. Opts:

[~] # df -k
Filesystem           1K-blocks      Used Available Use% Mounted on
none                    409600    146788    262812  36% /
devtmpfs                481384         0    481384   0% /dev
tmpfs                   131072        20    131052   0% /tmp
tmpfs                   500332         0    500332   0% /dev/shm
tmpfs                    16384         0     16384   0% /share
df: /mnt/snapshot/export: No such file or directory
cgroup_root             500332         0    500332   0% /sys/fs/cgroup
tmpfs                    24576         0     24576   0% /smb_tmp
tmpfs                    65536         0     65536   0% /tunnel_agent
/dev/vg1/lv1             10871     10625         0 100% /mnt

IT'S WORKS
```
now you can mount the other disk (on VM is /dev/sdb to a another mount copy and copy the data.



## test the qnap test volume on linux box
you can try to mount the test volume **ddrescue_volume.img** on a linux box to see what happen-

```
# losetup /dev/loop10 ddrescue_volume.img
# mdadm --examine /dev/loop10
/dev/loop10:
          Magic : a92b4efc
        Version : 1.0
    Feature Map : 0x0
     Array UUID : e767caf1:30504500:f7da4b48:92bb490e
           Name : 1
  Creation Time : Sat Oct 21 09:22:29 2023
     Raid Level : raid1
   Raid Devices : 2

 Avail Dev Size : 64216 sectors (31.36 MiB 32.88 MB)
     Array Size : 32064 KiB (31.31 MiB 32.83 MB)
  Used Dev Size : 64128 sectors (31.31 MiB 32.83 MB)
   Super Offset : 64232 sectors
   Unused Space : before=0 sectors, after=104 sectors
          State : active
    Device UUID : d2715d67:9bcc2c06:addcbc12:c70281b2

    Update Time : Sat Oct 21 18:21:20 2023
       Checksum : ef04233c - correct
         Events : 18


   Device Role : Active device 0
   Array State : AA ('A' == active, '.' == missing, 'R' == replacing)
# mdadm --assemble --run /dev/md99  /dev/loop10 --uuid=e767caf1:30504500:f7da4b48:92bb490e
mdadm: /dev/md99 has been started with 1 drive (out of 2).

# vgs
  WARNING: PV /dev/md99 in VG vg1 is using an old PV header, modify the VG to update.
  VG     #PV #LV #SN Attr   VSize    VFree
  cl       1   3   0 wz--n- <110.20g  33.57g
  datavg   1   1   0 wz--n-    1.36t 596.90g
  vg1      1   3   1 wz--n-   28.00m      0
# lvs
  WARNING: PV /dev/md99 in VG vg1 is using an old PV header, modify the VG to update.
  LV        VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home      cl     -wi-a----- <10.63g
  root      cl     -wi-ao----  50.00g
  swap      cl     -wi-ao----  16.00g
  datalv    datavg -wi-ao---- 800.00g
  lv1       vg1    owi---tz--  12.00m tp1
  snapShots vg1    swi---s---   4.00m      lv1
  tp1       vg1    twi---tz--  20.00m

# lvchange -ay vg1/lv1 -v
  WARNING: PV /dev/md99 in VG vg1 is using an old PV header, modify the VG to update.
  Activating logical volume vg1/lv1.
  activation/volume_list configuration setting not defined: Checking only host tags for vg1/lv1.
  Creating vg1-tp1_tmeta
  Loading table for vg1-tp1_tmeta (253:4).
  Resuming vg1-tp1_tmeta (253:4).
  Creating vg1-tp1_tdata
  Loading table for vg1-tp1_tdata (253:5).
  Resuming vg1-tp1_tdata (253:5).
  Executing: /usr/sbin/thin_check -q --clear-needs-check-flag /dev/mapper/vg1-tp1_tmeta
  /usr/sbin/thin_check failed: 1
  Check of pool vg1/tp1 failed (status:1). Manual repair required!
  Removing vg1-tp1_tmeta (253:4)
  Removing vg1-tp1_tdata (253:5)

```
it doesn't work. In other circumstances
```
  LV vg1/tp1, segment 1 invalid: does not support flag ERROR_WHEN_FULL. for tier-thin-pool segment.
  Internal error: LV segments corrupted in tp1.
  Cannot process volume group vg1
```



## notes
the tmp space is calculate on the ammount of ram assigned via qemu. if you need more space after the boot of the custom kernel you can do
```
mount -t tmpfs tmpfs /tmp -o remount,size=256M
```



