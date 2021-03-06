Welcome to the ZFS-root-filesystem-ubuntu wiki!

# <a name="requis">System Requirements</a>
* 64-bit Ubuntu at least 16.04 or newer boot/install CD/USB/DVD, full installer not server, netboot or alternative
* Need to use Legacy/BIOS we will update it to the EFI soon
* 5 disks or more completely wiped
* 12GB memory or more is recommended but it works for test purpose with 4GB

# <a name="sources">Sources</a>
We use this wiki as reference 
https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Ubuntu-16.04-to-a-Whole-Disk-Native-ZFS-Root-Filesystem-using-Ubiquity-GUI-installer/
but we use 5x 2TB Hardrive to do Raid6 "raidz2" for this tutorial 


# Strategy

The Ubiquity installer does not recognize the ZFS filesystem as a usable target, however it can be installed to a ZFS Zvol then manually copied to the ZFS filesystem.

The best practice is to use devices found in "/dev/disk/by-id/", not /dev/ when creating pools. Some prefer to use "wwn-" devices listed in this directory, however not all devices have these identifiers.  Please inventory what you have in your system and use the proper device names as you go along. In the examples below, we'll use 5 disks "/dev/disk/by-id/ata-ST8888888_2000000". 

# <a name="checkBeforeInstall">STEP 0 : Check this before installation</a>
* Do Boot on your USB ubuntu LTS
* For server installation set your IP manually and Remove NetworkManager
* Do "sudo apt-get update" if you have a problem with streamcli you have to do this in cli : "sudo apt-get upgrade"
* Do a "rm /dev/disk/by-id/*" to clean all the device ID
**Make sure to install "gparted" and make a refresh to reload the device ID (you can check with **ls /dev/disk/by-id/ -la**)

# <a name="prepareToInstall">STEP 1 : Prepare The Install Environment</a>
Launch gparted and create a new partition table and choose gpt on all of your disk which you want to add in zfs

# <a name="install">STEP 2 : Installation of zfs on the live/cd</a>
We choose raidz2 but you can take whatever you want but if you do a mirror you have to choose which boot into the kernel ...
Create the pool with ID (ls /dev/disk/by-id/ -la)
```
sudo su
apt-add-repository universe
apt-get update
apt-get install -y zfs-initramfs

## If you have small disk or if you'll need to change your disk for a bigger one, add -o autoexpand=on in the line below ##
zpool create -o ashift=12 -O atime=off -O compression=lz4 -O normalization=formD t5500 raidz2 /dev/disk/by-id/ata-TOSHIBA_DT01ACA200_46GR399KAS /dev/disk/by-id/ata-TOSHIBA_DT01ACA200_46NG799AS /dev/disk/by-id/ata-TOSHIBA_DT01ACA200_46NG599LAS /dev/disk/by-id/ata-TOSHIBA_DT01ACA200_46NG439VS /dev/disk/by-id/ata-TOSHIBA_DT05ACA200_46NG598LAS

zfs create -V 10G t5500/tmp

```

### If you need to install a spare disk
[Click here for add a spare disk](#spare)

# <a name="UbuntuGUI">STEP 3 : start ubuntu installer</a>
It's on your Desktop make sure to choose **"something else"** in **installation type.**
When you will see all the drive you just have to scroll down and select zd0 add a new partition table then you can clic on zd0p1 and clic on "+", Select EXT4 and mountpoint=/ In the Bootloader dropdown, select "/dev/zd0" Press "Install Now"

Additional Screens will come up such as timezone and user account creation. Complete these with your information. Eventually you will get an error about the bootloader not being able to be installed. Choose "Continue without a bootloader" At the end of the install select "Continue testing"
# <a name="copy">Copy your Ubuntu image to the ZFS filesystem</a>
***Create all the dataset you need before the rsync***
```
mount /dev/zd0p1 /mnt
zfs create t5500/ROOT
zfs create t5500/ROOT/ubuntu
# rsync -avPX /mnt/. /t5500/ROOT/ubuntu/.
```
# <a name="mount">STEP 4 : Mount the filesystem and change the grub parameter</a>
```
for d in proc sys dev; do mount --bind /$d /t5500/ROOT/ubuntu/$d; done
# chroot /t5500/ROOT/ubuntu
echo "nameserver 8.8.8.8" | tee -a /etc/resolv.conf
apt-get update
apt install -y openssh-server vim net-tools zfs-initramfs ifenslave #if you want to do a [bonding](#bondingSetup)
## You can add your ssh keys into ***/root/.ssh/authorized_keys***
apt remove network-manager -y ## network is more stable with a setup in /etc/network/interfaces
# nano /etc/default/grub       ## Find GRUB_CMDLINE_LINUX= and add "boot=zfs rpool=t5500 bootfs=t5500/ROOT/ubuntu"
```
# <a name="prepareGrub">STEP 5 : Prepare the grub partition</a>
We need to link the partition with device because if we don't the grub won't see the partition and ***you have to do it on all of your disk***
***Be careful with the links (ln) if you failed one use unlink /dev/deviceWhichFailed***
```
# ***Do it on all of your disk***
sgdisk -a1 -n2:512:2047 -t2:EF02 /dev/disk/by-id/ata-TOSHIBA_DT01ACA200_46GR399KAS
#***IF YOU HAVE UBUNTU 16.04 please do this for each partition you have so the partition link to his disk***
#***IF YOU HAVE UBUNTU newer version you don't need to do the links on each partition go directly to update-grub***
ln -sf /dev/ata-TOSHIBA_DT01ACA200_46GR399KAS-part1 /dev/ata-TOSHIBA_DT01ACA200_46GR399KAS
# update-grub
# ***do it on all of your disk ***
# grub-install /dev/disk/by-id/ata-TOSHIBA_DT01ACA200_46GR399KAS
```
# <a name="reboot">STEP 6 : Set Mountpoint and reboot
```
exit
umount -R /t5500/ROOT/ubuntu
zfs set mountpoint=/ t5500/ROOT/ubuntu
#if you have home dataset you need do this mountpoint otherwise you can create a dataset like t5500/ROOT/ubuntu/home
zfs set mountpoint=/ data/home  
zfs snapshot t5500/ROOT/ubuntu@pre-reboot
umount /mnt
zpool export -a
# reboot
```
If you get an error saying that the pool cannot be exported, run the command below:
```
ltrace -f zpool export t5500
```

# <a name="finish">STEP 7 : Almost finish just a few commands</a>
After your reboot on the system you just have to update the system and make a new a snapshot if you need to recover you can use zpool rollback t5500/@nameOfYourSnapshot
To make a list of all your snapshot : zfs list -t snapshot
```
sudo zfs snapshot t5500/ROOT/ubuntu@post-reboot
sudo zfs destroy t5500/tmp

sudo zfs snapshot t5500/ROOT/ubuntu@post-reboot-updates
zpool scrub t5500

# WAIT that scrub finish you can check the process with zpool status and then reboot
reboot
# ZFS use a ARC fonction which use a lot of memory, we advice everyone to create a swap to reduce the use of the memory, we will use a ZVOL to create the swap we will need (0.5% of the total RAM)
zfs create -V 1G t5500/ROOT/swap
mkswap /dev/zvol/t5500/ROOT/swap
swapon /dev/zvol/t5500/ROOT/swap
#the free command will give the amount of RAM from the computer
free
root@adminimt:~# free
              total        used        free      shared  buff/cache   available
Mem:        2034864      830664      969032        1660      235168     1058112
Swap:       1048572           0     1048572

#after you create the swap partition you will need to add the line below to the /etc/fstab
/dev/zvol/t5500/ROOT/swap none swap defaults 0 0

# If you want to upgrade to 16.10 please follow this instruction
# ***Do this for each partition you have so the partition with his disk***

ln -sf /dev/ata-TOSHIBA_DT01ACA200_46GR399KAS-part1 /dev/ata-TOSHIBA_DT01ACA200_46GR399KAS
# If you got an error please check if the links are correct
sudo apt update
sudo apt dist-upgrade

```
Now you have finish
# <a name="maintenance">Maintenance</a>
## Replace Disk
set offline the defective disk and then replace it, if you want to use the new disk as a bootable disk you have to create a partition and recreate the links then you can install grub
```
zpool offline t5500 /dev/disk/by-id/oldDisk
zpool replace t5500 /dev/disk/by-id/OldDisk /dev/disk/by-id/newDisk
# bootable disk
# create a partition for grub
sgdisk -a1 -n2:512:2047 -t2:EF02 /dev/disk/by-id/newDisk
# ***Do this for each partition you have so the partition with his disk, this link is used to install grub on zfs***
ln -sf /dev/ata-TOSHIBA_DT01ACA200_46GR399KAS-part1 /dev/ata-TOSHIBA_DT01ACA200_46GR399KAS #for example here is my sda disk
ln -sf /dev/ata-WDC_WD5000AAKS-60Z1A0_WD-WCAWF6072225-part1 /dev/ata-WDC_WD5000AAKS-60Z1A0_WD-WCAWF6072225 #disk B
ln -sf /dev/ata-WDC_WD5000AAKX-603CA0_WD-WMAYUV189661-part1 /dev/ata-WDC_WD5000AAKX-603CA0_WD-WMAYUV189661 #disk C etc...

update-grub
grub-install /dev/disk/by-id/newdisk
zpool scrub t5500
# wait a few seconds, you can check the progression with 'zpool status
```
# <a name="troubleshot">Troubleshoot</a>
## Grub patch
https://launchpad.net/~cyphermox/+archive/ubuntu/ppa
```
sudo add-apt-repository ppa:cyphermox/ppa
sudo apt-get update
```
## <a name="spare">Install a spare disk</a>
```
zpool add tank spare /dev/disk/by-id/*disk_id*
```
## Network
# <a name="bondingSetup">Bonding setup</a>
before to this commands please check if your are using network-manager
```
nmcli dev status
```
to disable Network Manager on Ubuntu :
```
sudo service network-manager stop
sudo service network-manager disable

```
For ubuntu 18.04.2
```
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
```
Remove Network Manager 
```
systemclt stop networkmanager.service
systemclt disable networkmanager.service
apt remove --purge network-manager
```
Now we can procedd with bonding setup
```
sudo modprobe bonding
sudo apt install ifenslave #if you do not want to install ifenslave follow [this procedure](#withoutIfenslave) 

Add the network configuration

nano /etc/network/interfaces

auto lo eno1 eno2 bond0
iface lo inet loopback

iface eno1 inet manual
    bond-master bond0

iface eno2 inet manual
    bond-master bond0

iface bond0 inet static
        address 192.168.1.45.
        netmask 255.255.255.0
        gateway 128.178.242.1

We can now create the bonding interface and add the network interface as slaves for it       

ip link add bond0 type bond mode 802.3ad
ip link set eno1 down && ip link set eno2 down      
ip link set eno1 master bond0 
ip link set eno2 master bond0

Check you DNS server

/etc/resolv.conf
nameserver 8.8.8.8

We can restart the network and test it

service networking restart
ping 8.8.8.8
  
Check to see if fail-over now works by removing eno1 from bond0:

Check if bonding  is enabled
lsmod |grep bond
remove one interface and check if you can ping 8.8.8.8
ifenslave -d eno1 bond0
add it back
ifenslave eno1 bond0
```
Resolv.conf reset at each reboot
```
If your resolv.conf file is reset for each reboot please feel free to follow this procedure

sudo rm -f /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf

edit nano /etc/systemd/resolved.conf, adding your desired DNS server:

change this:
[Resolve]
#DNS=

to this (but use the one you want - this is an example):
[Resolve]
DNS=192.168.1.152

after that, restart the service:
service systemd-resolved restart
then reboot

```

```
OR you can just remove it and just remove the /etc/resolv.conf Or /run/resolv.conf files then you can create a new /etc/resolv.conf and add you dnsserver like this
```
nameserver 8.8.8.8
```
then you can change the network configuration in "/etc/network/interface"

## Apt-get update failed streamcli
 
```
sudo apt-get upgrade
sudo apt-get update
```

## Grub-probe:error:failed to get canonical path of /cow.
Your links isn't correct so you can unlink them and recreate again

```
ln -sf /dev/diskA-part1 /dev/diskA
ln -sf /dev/diskB-part1 /dev/diskB
ln -sf /dev/diskC-part1 /dev/diskC
update-grub
grub-install /dev/disk/by-id/diskA
grub-install /dev/disk/by-id/diskB
grub-install /dev/disk/by-id/diskC
```

## Chroot
Please do your network configuration first then follow the steps below

```
sudo apt install zfs-initramfs -y
zpool import -a
zfs set mountpoint=/t5500/ROOT/ubuntu t5500/ROOT/ubuntu
zfs mount -a
for d in proc sys dev; do mount --bind /$d /t5500/ROOT/ubuntu/$d; done
chroot /t5500/ROOT/ubuntu
exit
umount -R /t5500/ROOT/ubuntu
zfs set mountpoint=/ t5500/ROOT/ubuntu
zpool export t5500
reboot
```

## ** (appstreamcli:7091): CRITICAL **: Error while moving old database out of the way. 

```
Solution :
chmod g+w /var/cache/app-info/xapian/default -R
```

