On tty

/etc/init.d/sshd start    

ssh onto machine    

## configure disks

cfdisk /dev/sda

1 &emsp; efi  &emsp; &emsp;1GB  
2 &emsp; ext2 &emsp; 62GB  
2 &emsp; swap &emsp;1GB

mkfs.vfat -F 32 /dev/sda1         
mkfs.ext4 /dev/sda3

mkswap /dev/sda2      
swapon /dev/sda2

mount /dev/sda3 /mnt/gentoo

## download Stage 3

cd /mnt/gentoo

links    
gentoo.org     
choose arm stage3 openrc     

tar xpvf <downloaded stage 3 file> --xattrs-include='*.*' --numeric-owner

nano /etc/portage/make.conf    

MAKEOPTS="-j2"  
ACCEPT_LICENSE="-* @FREE @BINARY-REDISTRIBUTABLE"  
GRUB_PLATFORMS="efi-64"  
CPU_FLAGS_ARM="aes sha3 crc32 neon v8 vfpv4"  


## Enter chroot

cp /etc/resolv.conf /mnt/gentoo/etc/

mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

chroot /mnt/gentoo /bin/bash
. /etc/profile

mount /dev/sda1 /boot

## uodate packages

emerge-webrsync
emerge --sync
emerge --ask --verbose --update --deep --newuse @world

## timezone and locale

echo "America/Vancouver" > /etc/timezone
emerge --config sys-libs/timezone-data

sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen

eselect locale list
eselect locale set <locale>

echo "LANG=en_US.UTF-8" >> /etc/locale.conf

## Configure Kernal

echo "sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE" | tee -a /etc/portage/package.license

emerge --ask sys-kernel/linux-firmware
emerge --ask sys-kernel/gentoo-sources

eselect kernel list
eselect kernel set 1

cd /usr/src/linux
make menuconfig <-- save file

make -j2 Image
make install
make modules
make modules_install

## Set up DHCP and network

emerge dhcpcd
rc-update add dhcpcd default

nano /etc/conf.d/net
config_enp0s5="dhcp"

cd /etc/init.d
ln -s net.lo net.enp0s5
rc-update add net.enp0s5 default

## install bootloader 

emerge sys-boot/efibootmgr
emerge --verbose sys-boot/grub
grub-install --target=arm64-efi --efi-directory=/boot --bootloader-id=Gentoo
grub-mkconfig -o /boot/grub/grub.cfg

## Set password

passwd

## set up NTP 

emerge --ask net-misc/ntp
rc-update add ntp-client default

## edit fstab

nano /etc/fstab

/dev/sda1   /boot        vfat    umask=0077     0 2
/dev/sda2   none         swap    sw                   0 0
/dev/sda3   /            ext4    defaults,noatime              0 1
/dev/cdrom  /mnt/cdrom   auto    noauto,user          0 0

## configure hostname 

echo gentoo > /etc/hostname

## reboot system

exit
cd
umount -R /mnt/gentoo
reboot




POST

Log onto tty

nano /etc/ssh/sshd_config <-- password to yes
/etc/init.d/sshd start
rc-update add sshd default

Log onto ssh

emerge --ask sys-kernel/dracut
dracut

nano /etc/fstab

/dev/sda1   /boot        vfat    umask=0077     0 2
/dev/sda2   none         swap    sw                   0 0
/dev/sda3   /            ext4    defaults,noatime              0 1
/dev/cdrom  /mnt/cdrom   auto    noauto,user          0 0

echo gentoo > /etc/hostname

sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen

eselect locale list
eselect locale set <locale>

echo "LANG=en_US.UTF-8" >> /etc/locale.conf

set up NTP

nano /etc/conf.d/hwclock
clock="local"

Fix clock skew
cd /
touch fixtime
find . -cnewer /fixtime -exec touch {} \; <-- ignore errors

emerge --ask net-misc/ntp
rc-update add ntp-client default
