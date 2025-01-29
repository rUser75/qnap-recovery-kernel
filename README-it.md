# qnap-recovery-kernel
un kernel QTS minimo, con supporto lvm, in esecuzione su qemu


## Introduzione
Qualche tempo fa ho aiutato un mio amico a recuperare i dati dal suo QNAP che non si accendeva più.

il volume di dati era su un volume LVM creato su un raid md, questo è lo standard per i dispositivi QNAP (almeno per quelli a due dischi).

il mio primo tentativo è stato quello di collegare uno degli hdd su un computer linux per montare il volume ma sono rimasto sorpreso nello scoprire che non sono in grado di farlo. Perché?
QNAP non utilizza un LVM Linux standard ma una compilazione personalizzata leggermente diversa ed è per questo che il software di recupero dati non ha funzionato.

Dopo molto tempo abbiamo acceso il vecchio QNAP con un altro HDD e siamo riusciti a recuperare i dati presenti sul vecchio HDD.
ma... se il QNAP è definitivamente rotto?
per questo motivo ho deciso di assemblare questo kernel partendo da un firmware di un dispositivo qnap x86

## prima di iniziare
Se il NAS ha due dischi prima dell'avvio è importante contrassegnare i dischi per ricordare la posizione corretta.
per me è importante anche non lavorare direttamente sui dischi ma su una copia.
la copia della partizione dati può essere eseguita utilizzando uno strumento come dd o ddrescue.
> ** Se continui a leggere suppongo che tu sappia usare Linux e che non ci siano garanzie che tutto funzioni come previsto...**


## crea una copia della partizione dati
collegare uno dei dischi a un sistema Linux (è possibile utilizzare un adattatore SATA-USB, un box USB, collegato direttamente...)
usando fdisk per determinare la partizione dati, ad esempio quella grande (solitamente la partizione 3)
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


dopo aver identificato la partizione giusta, utilizzando dd o ddrescue puoi  salvare il file immagine su di un'altro disco

```
dd if=/dev/sda3 of=/otherstoragepath/sda3.img bs=8k 
```

oppure

```
ddrescue -n /dev/sda3 /otherstoragepath/sda3.img /pathtologfile/logfile
```
> **nota** usando ddrescue puoi provare a correggere alcuni errori I/O se l'hdd di origine è stato danneggiato e specificando il file di registro (ora chiamato domain-mapfile) puoi interrompere e riprendere la copia


Di seguito i comandi di base per installare ddrescue su centos
```
dnf -y install ddrescue
```
o su ubuntu
```
add-apt-repository universe
apt-get install gddrescue
```

## prova a recuperare.
hai bisogno di un box linux (centos, ubuntu live sono testati) con sufficiente spazio libero per copiare i dati recuperati (con un disco usb esterno ad esempio)

prima di iniziare controlla se i pacchetti qemu sono installati, altrimenti installali usando
```
dnf install -y qemu-kvm (per centos)

apt-get install qemu-kvm (per ubuntu)
```


Scaricare le immagini del kernel allegate qui e metterle in una directory.

Nel menu di rilascio (https://github.com/rUser75/qnap-recovery-kernel/releases/tag/first) puoi trovare un
* initrdC.boot più grande
* file di prova lvm
  
  
eseguire il seguente comando per eseguire il kernel personalizzato in una macchina virtuale su RedHat (su Ubuntu sostituire /usr/libexec/qemu-kvm con qemu-system-x86_64)

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

> sdb3.img è il nome del file immagine o se vuoi usare il disco il dispositivo a blocchi del disco (/dev/sdXXXX)

> /dev/disk_were_copy è il disco esterno in cui puoi copiare i dati

**nota:** all'interno della VM la prima riga -drive diventa /dev/sda la seconda /dev/sdb ecc.

Se tutto funziona come previsto, la macchina virtuale si avvia.
```
Welcome to the recovery kernel for QNAP product

use admin/admin for login

NAS login:
```

ora possiamo provare a montare il volume aperto (puoi usare ddrescue_volume.img per il test)

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

FUNZIONA!!
```

Ora puoi montare l'altro disco (sulla VM è /dev/sdb) su un'altra copia montata e copiare i dati.



## testare il volume di prova qnap sulla casella linux
puoi provare a montare il volume di prova **ddrescue_volume.img** su una casella Linux per vedere cosa succede-

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

non funziona. In altre circostanze
```
  LV vg1/tp1, segment 1 invalid: does not support flag ERROR_WHEN_FULL. for tier-thin-pool segment.
  Internal error: LV segments corrupted in tp1.
  Cannot process volume group vg1
```



## note
lo spazio tmp viene calcolato sulla quantità di ram assegnata tramite qemu. se hai bisogno di più spazio dopo l'avvio del kernel personalizzato puoi farlo
```
mount -t tmpfs tmpfs /tmp -o remount,size=256M
```



   


