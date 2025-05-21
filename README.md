# wd-pr4100-ubuntu
Original link: [WD Forums](https://community.wd.com/t/guide-how-to-install-ubuntu-18-04-server-on-the-my-cloud-pr4100-nas/232786).
Thank you: Tfl

# Running Ubuntu on WD My Cloud PR4100
Requirements:

    a PC running Ubuntu 18 or similar… and some experience with unix.
    USB flash drive (8GB+)
    WD My Cloud PR2100/PR4100

If you’re in a hurry, skip the guide and go straight for the boot image at the end of the post.

## Get KVM

`sudo apt install qemu-kvm ovmf`

## Prepare a working directory

`mkdir ubuntu && cd ubuntu`

Copy the UEFI bootloader to a local file named bios.bin

`cp /usr/share/ovmf/OVMF.fd bios.bin`

## Download the Ubuntu 18.04.1 server iso.

Find out the name of your USB flash drive with lsblk. I’ll use /dev/sdX here.
Boot the iso installer.
```
sudo kvm -bios ./bios.bin -cdrom <path_to_iso> \ 
         -drive format=raw,file=/dev/sdX -boot once=d -m 1G`
```

It should mention TIANO CORE during boot and then enter a black grub screen to install Ubuntu.
Complete the installation with the defaults and any extra package that you may be interested in (e.g. Nextcloud).
Note down the user and password, you need it to login into the machine later.
At the end, it will reboot and ask you to remove the cdrom.
Just close the whole window to shutdown the whole virtual machine.
Then boot without cdrom straight from the USB flash drive.

`sudo kvm -bios ./bios.bin -drive format=raw,file=/dev/sdX -m 1G`

Login in the virtual machine and update packages if you like.

## Networking
Ubuntu is now installed for a virtual network interface with the new udev persistent networking naming.
You’ll see the current network interface is called ens3 or similar.

`ip addr show`

This won’t work on actual My Cloud hardware.
Create the netplan configuration as follows

`sudo editor /etc/netplan/01-netcfg.yaml`
```
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: true
    eno2:
      dhcp4: true`
```

This causes the NAS to get a dynamic IPv4 address on both of its onboard (eno) interfaces.
Example with static IP and bonding

Here’s how to combine the throughput of the 2 network interfaces on a single static IP address.

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      optional: true
    enp2s0:
      dhcp4: false
      optional: true
  bonds:
    bond0:
      interfaces: [enp1s0, enp2s0]
      addresses: [192.168.0.100/24]
      gateway4: 192.168.0.1
      nameservers:
       addresses: [8.8.8.8,8.8.4.4]
      parameters:
        mode: 802.3ad
        mii-monitor-interval: 1
```

More info (static IP, bonding, …) on https://netplan.io

## Hardware Control
Thanks to the research of Michael Roland and @dswv42 we now have full control over the fan, lcd, buttons and sensors. Ubuntu ships with the 8250_lpss module, so you don’t need to build a custom kernel.
The PMC is accessible at serial port /dev/ttyS5.
You need some packages from the universe repo.

```
sudo add-apt-repository universe
git clone https://github.com/WDCommunity/wdnas-hwtools
cd wdnas-hwtools
sudo ./install.sh
```

The Ubuntu boot disk is now ready. Shutdown with

`sudo halt -p`

and plug the USB drive in the PR4100 NAS.
Boot up and enjoy!

## Extras
Download example boot disk image

Download my image here, unzip (use 7zip on winwods) and burn to a 16GB+ flash drive.
Direct unzip and write with

cat foo.img.gz | gunzip | dd of=/dev/sdX

Login: wdnas
Password: mycloud

Warning: risk for data loss when using the wrong device
To grow the file system to use the complete usb boot drive, delete the second partition.

sudo sgdisk /dev/sdX --delete=2

Create it again, it will automatically use all available space.

sudo sgdisk /dev/sdX --new=2

Refresh the partitions

sudo partprobe

Now that the partition has been resized, you can grow the file system. This is done on the second partition /dev/sdX2, not on the disk /dev/sdX.

sudo e2fsck -f /dev/sdX2
sudo resize2fs /dev/sdX2

###  Create a new ZFS array

Here’s a great overview on the core features of ZFS. [ZFS Link](https://wiki.archlinux.org/index.php/ZFS/Virtual_disks)
Now let’s create a ZFS array on the PRx100.

Insert your disks (hotplug is allowed). List them.
```
wdnas@wdnas:~$ lsblk -d
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0          7:0    0 86.9M  1 loop /snap/core/4917
loop1          7:1    0 89.5M  1 loop /snap/core/6130
sda            8:0    0  1.8T  0 disk 
sdb            8:16   0  1.8T  0 disk 
sdc            8:32   0  1.8T  0 disk 
sdd            8:48   0  1.8T  0 disk 
sde            8:64   1 14.3G  0 disk 
mmcblk0      179:0    0  3.7G  0 disk 
mmcblk0boot0 179:8    0    2M  1 disk 
mmcblk0boot1 179:16   0    2M  1 disk 
```
Here we see sde is the USB boot disk.
Create a mirror pool over sda and sdbbased on the [Ubuntu Tutorial](https://wiki.archlinux.org/index.php/ZFS/Virtual_disks). 
It’s recommended to name your pool media.

`sudo zpool create media mirror sda sdb`

Alternatively, create a raidz pool over 4 disks. This is similar to a RAID5 pool, using 1 disk for parity.

`sudo zpool create media raidz sda sdb sdc sdd`

In order to use it, you need to create a file system (also called dataset) on the zpool. This is similar to a ‘share’ in the My Cloud OS.
Here’s an example

`sudo zfs create media/pictures`

The file system gets mounted automatically at /media/pictures.

### or import existing data
SSH into My Cloud NAS running Ubuntu.
``
cat /proc/mdstat
sudo mdadm --assemble --scan
``
or if you used my FreeNAS image to create a ZFS array
```
sudo apt install zfsutils-linux
sudo zpool import
```
Follow the instructions.
It’s recommended to name your zpool ‘media’ so that the Ubuntu snap system can easily use it.

### Prevent disks from spinning up every 10 minutes

Disable udisks2

### Setup Nextcloud

I couldn’t explain it better than this Digital Ocean guide.
All data will be stored on the USB boot disk, which is not so interesting.
Here’s how to change the nextcloud snap data partition to your zfs pool.

First create a zfs dataset for nextcloud.

`sudo zfs create media/nextcloud`

Allow the nextcloud snap to use the zpool

`sudo snap connect nextcloud:removable-media`

Change the data path in /var/snap/nextcloud/current/nextcloud/config/autoconfig.php

`sudo sed -i "s#'directory' => .*#'directory' => '/media/nextcloud/data',#" /var/snap/nextcloud/current/nextcloud/config/autoconfig.php`

### Restart nextcloud

`sudo snap restart nextcloud.php-fpm`

Now visit your nextcloud website and create the admin user.
Disable internal flash memory

If the internal flash memory is completely broken, you may be unable to restore the origal WD OS.
Installing Ubuntu is a solution, but you’ll see system freezes when polling the disks in the dmesg output.
A solution is to blacklist the mmc_block driver.

`sudo editor /etc/modprobe.d/blacklist.conf`

Add a line with

`blacklist mmc_block`

Then

`sudo update-initramfs -u`



## TLDR
### Info 1
Spindown of the disks can be managed with [hd-idle](https://withblue.ink/2016/07/15/what-i-learnt-from-using-wd-red-disks-to-build-a-home-nas.html).
The disks spin when you access them of course… depends a bit what you install on them.
An external OS disk (on USB) is low noise anyway.

If you really need a GUI, you can install webmin. Here’s an overview. But as I said in the opening post… I don’t use a GUI.
Or see How do you run Ubuntu Server with a GUI? - Ask Ubuntu
I’d argue that nextcloud provides sufficient web gui once it’s installed.

You can safely test running Ubuntu from USB media, even access your data after import with mdadm, install nextcloud snap on USB … and when you’re not satisfied, shutdown and unplug the USB boot disk to go back to WD OS.
### Info 2



    Why the default FAN speed is at 810?
    When does it increase and what is the interval? - Is there any configuration file?
    Is the FAN increased on high CPU temp?
    What is your CPU temp? On the high load I have seen 60-65C.

I suggest you skim through these files

    wdhwdaemon/daemon.py
    wdhwlibs/temperature.py
    wdhwlibs/fancontroller.py
    tools/wdhwd.conf

Feel free to modify it to your needs… but I am quite happy with the current behaviour.
Note that CPU temperature is usually 20 degrees higher than the disks and you need to ensure some minimal airflow in such a condensed box (hence the minimal rpm).

Now I want to build the img myself.

        sudo add-apt-repository “deb Index of /ubuntu $(lsb_release -sc) universe”
        git clone GitHub - WDCommunity/wdnas-hwtools: Hardware Controller for Western Digital My Cloud x64 NAS Systems
        cd wdnas-hwtools
        sudo ./install.sh

    Is this above part done in VM?

No I usually install the wdhw tools only on the box itself. The reason is the automatic shutdown in the wdhw tools when it detects unusual temperatures (or fails to get the temperature). The VM doesn’t have a PMC module to talk to, so the wdhwd may panic and go in a bootloop.

    Are both ethernet cables plugged? Mine was only one cable, but it didn’t get IP. Then I inserted second cable, eventually it worked, but on LCD I got IP printed as 999 while the original IP was 192.168.1.22

The LCD scripts are in /usr/local/lib/wdhwd/scripts/show_lcd.sh
There too many options here so you need to adapt it to your needs.
Feel free to share your scripts though!

When using a static IP, you may need to set the default gateway and DNS servers too, as these normally come from the DHCP server.
You may also use bonding to get a single IP valid for both network interfaces (like the WD firmware does by default). Just check the netplan.io examples.

### How to replace your ZFS pool drives

This procedure can be used to replace faulty disks or to upgrade all your disks (one by one) to increase your zpool size. Always make a backup of your data first and consider using a UPS.
First backup your current configuration. The zdb command provides detailed info on your pool. It comes in handy when the disk is gone from your system.
`zdb > media.zdb`
Disable the first drive from the pool. The pool is named media. My pool does not spread over the complete disk, only he second partition of each disk is used.
`zpool offline media /dev/sda2`
Remove the drive from the slot and insert a new drive (with at least the same capacity).

I had to create a partition table first. I simply copied the partition table of one of the other disks (sdb) and generated new disk GUIDs. You don’t need to do this if your pool uses the complete disk.
```
sgdisk --replicate=/dev/sda /dev/sdb
sgdisk -G /dev/sda
partprobe
```
Then resilver the pool with
`zpool replace media /dev/sda2 /dev/sda2`
Track the resilver progress with
`zpool status media`
It took about 15 hours for a full 4TB WD red drive.
