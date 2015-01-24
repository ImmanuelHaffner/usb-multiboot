Setting up a bootable USB drive with multiple OS using Syslinux
=====

The USB drive will have only a single partition. (I have tried using multiple
partitions and chainloading them, but it did not work.)
For the remainder of this guide, I will refer to the USB device with
`/dev/sdX` and to the first (and only) partition on this device as `/dev/sdX1`.

## Prepare the USB drive

Make sure the USB device is __not__ mounted.

### Partitioning

We start with partitioning our USB device.  Run `fdisk` on the device.

```
# fdisk /dev/sdX

Welcome to fdisk (util-linux 2.25.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
```

When prompted for a command, create a new __DOS partition table__.
(Syslinux requires DOS partitons.)

```
Command (m for help): o
Created a new DOS disklabel with disk identifier 0x9c4e2fab.
```

We now have an empty DOS partition table.  Now create a single partition that
spans the whole device.  After typing `n` just press *ENTER* to use the default
options.

```
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 
Partition number (1-4, default 1): 
First sector (2048-30310399, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-30310399, default 30310399): 

Created a new partition 1 of type 'Linux' and of size 14.5 GiB.
```

To verify this procedure, enter `p` to print the current partition table.

```
Command (m for help): p
Disk /dev/sdX: 14.5 GiB, 15518924800 bytes, 30310400 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x9c4e2fab

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdX1        2048 30310399 30308352 14.5G 83 Linux
```

Now we have to change the type of the partition to `W95 FAT32`.  I am not sure
whether the Hex code is the same for every system, so use `L` to print the table
of Hex codes first.

```
Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): L
0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris        
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden C:  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx         
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data    
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility   
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt         
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access     
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O        
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor      
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi eb  BeOS fs        
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         ee  GPT            
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ef  EFI (FAT-12/16/
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        f0  Linux/PA-RISC b
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f1  SpeedStor      
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f4  SpeedStor      
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f2  DOS secondary  
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      fb  VMware VMFS    
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fc  VMware VMKCORE 
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fd  Linux raid auto
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fe  LANstep        
1c  Hidden W95 FAT3 75  PC/IX           be  Solaris boot    ff  BBT            
1e  Hidden W95 FAT1 80  Old Minix      
Hex code (type L to list all codes): b
(There might be another line of output... not imprtant)
Changed type of partition 'Linux' to 'W95 FAT32'.
```

To make the partition bootable, we have to set its boot flag.

```
Command (m for help): a
Selected partition 1
The bootable flag on partition 1 is enabled now.
```

Finally we write the changes to the device.

```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```


## Create the filesystem

We now create a MS-DOS FAT32 filesystem on the partition.
Replace `<LABEL>` by a name of your choice.

```
# mkfs.fat -F32 -n <LABEL> /dev/sdX1
mkfs.fat 3.0.27 (2014-11-12)
```

If there is a warning/error in the output, try again with the `-c` option.

```
# mkfs.fat -F32 -c -n <LABEL> /dev/sdX1
```


## Install Syslinux bootloader on the device

Mount the device.

```
# mkdir /mnt/usb
# mount /dev/sdX1 /mnt/usb
```

Create a directory for syslinux.

```
# mkdir -p /mnt/usb/boot/syslinux
```


### Copy required c32 modules

Copy necessary c32 modules.  (I am a lazy person, I just copy all the c32
modules.  One could identify the actually used modules, but since storage
capacity is not an issue...)
The location of the c32 modules depends on your OS and Syslinux installation.

```
# whereis syslinux
syslinux: /usr/bin/syslinux /usr/lib/syslinux /usr/share/man/man1/syslinux.1.gz
```

In my case they can be found at `/usr/lib/syslinux`.  Now copy the required
modules.

```
# cp /usr/lib/syslinux/bios/*.c32 /mnt/usb/boot/syslinux
```


### Install Syslinux

Install Syslinux on the partition.  With the `--directory` option we specify a
relative path to where the Syslinux binary is placed.

```
# syslinux --directory boot/syslinux --install /dev/sdX1
```


### Write MBR

To make the device bootable, we need to write the Master Boot Record of the
device.

```
# dd bs=440 count=1 conv=notrunc if=/usr/lib/syslinux/bios/mbr.bin of=/dev/sdX
1+0 records in
1+0 records out
440 bytes (440 B) copied, 8.7927e-05 s, 5.0 MB/s
```


### Deploy a sample configuration and test

To check that nothing went wrong to this point, we deploy a sample Syslinux
configuration file.

Create the file `/mnt/usb/boot/syslinux/syslinux.cfg` and open it with your
favorite text editor.

```
# vim /mnt/usb/boot/syslinux/syslinux.cfg
```

Paste the following lines of code to the file, then save and exit.

```
DEFAULT hdt
PROMPT 0
TIMEOUT 50

# You can create syslinux keymaps with the keytab-lilo tool
#KBDMAP de.ktl

UI vesamenu.c32
MENU TITLE Multitool
MENU RESOLUTION 640 480

MENU WIDTH 110
MENU MARGIN 35
MENU ROWS 5
MENU VSHIFT 8

MENU TIMEOUTROW 13
MENU TABMSGROW 11
MENU CMDLINEROW 11
MENU HELPMSGROW 20
MENU HELPMSGENDROW 23

# Refer to http://www.syslinux.org/wiki/index.php/Comboot/menu.c32

MENU COLOR border       30;44   #40ffffff #a0000000 std
MENU COLOR title        1;36;44 #9033ccff #a0000000 std
MENU COLOR sel          7;37;40 #e0ffffff #20ffffff all
MENU COLOR unsel        37;44   #50ffffff #a0000000 std
MENU COLOR help         37;40   #c0ffffff #a0000000 std
MENU COLOR timeout_msg  37;40   #80ffffff #00000000 std
MENU COLOR timeout      1;37;40 #c0ffffff #00000000 std
MENU COLOR msg07        37;40   #90ffffff #a0000000 std
MENU COLOR tabmsg       31;40   #30ffffff #00000000 std


LABEL hdt
  MENU LABEL HDT (Hardware Detection Tool)
  COM32 hdt.c32

LABEL reboot
  MENU LABEL Reboot
  COM32 reboot.c32

LABEL poweroff
  MENU LABEL Poweroff
  COM32 poweroff.c32
```

Now unmount the device.

```
# umount /mnt/usb
```

Leave the USB drive plugged in. Restart your computer and make sure that the USB
drive has the highest boot priority.
The USB device should be booted, and you are shown the following table:

```
+---------------------------------------------+
|                                             |
|                  Multitool                  |
|                                             |
+---------------------------------------------+
|                                             |
|  HDT (Hardware Detection Tool)              |
|                                             |
|  Reboot                                     |
|                                             |
|  Poweroff                                   |
|                                             |
+---------------------------------------------+
```

If this is not the case, make sure you followed the guide correctly.  If it
still does not work, feel free to open a new Issue and explain in detail what is
going wrong.
