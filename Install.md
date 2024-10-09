## Config VM

4GB Ram  
2 CPU's  
64GB Disk  

## Start SSH 

/etc/init.d/sshd start      
passwd

ssh onto machine    

## Configure disks

cfdisk /dev/sda

1 &emsp; efi  &emsp; &emsp;1GB  
2 &emsp; ext2 &emsp; 62GB  
2 &emsp; swap &emsp;1GB

mkfs.vfat -F 32 /dev/sda1         
mkfs.ext4 /dev/sda3

mkswap /dev/sda2      
swapon /dev/sda2

mount /dev/sda3 /mnt/gentoo

## Download Stage 3

cd /mnt/gentoo

links    
gentoo.org     
choose arm stage3 openrc     

tar xpvf st\* --xattrs-include='\*.\*' --numeric-owner

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

## Update packages

emerge-webrsync  
emerge --sync  
emerge --verbose --update --deep --newuse @world  

## Timezone and Locale

echo "America/Vancouver" > /etc/timezone  
emerge --config sys-libs/timezone-data

sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen  
locale-gen

eselect locale list  
eselect locale set 4  <-- check that 4 is correct

echo "LANG=en_US.UTF-8" >> /etc/locale.conf

## Configure Kernel

echo "sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE" | tee -a /etc/portage/package.license

emerge sys-kernel/linux-firmware  
emerge sys-kernel/gentoo-sources

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

echo 'config_enp0s5="dhcp"' > /etc/conf.d/net  

cd /etc/init.d  
ln -s net.lo net.enp0s5  
rc-update add net.enp0s5 default

## Set password

passwd

## Set up Chrony 

emerge net-misc/chrony  
rc-update add chronyd default  

## Edit fstab

nano /etc/fstab

/dev/sda1   /boot        vfat    umask=0077     0 2  
/dev/sda2   none         swap    sw                   0 0  
/dev/sda3   /            ext4    defaults,noatime              0 1  
/dev/cdrom  /mnt/cdrom   auto    noauto,user          0 0  

## Configure hostname 

echo gentoo > /etc/hostname

## Disable F0

sed -i 's/f0:/#f0:/' /etc/inittab  

## Enable SSHD

sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config   

rc-update add sshd default

## Option 1 - Install bootloader (GRUB)

emerge sys-boot/efibootmgr  
emerge --verbose sys-boot/grub  
grub-install --target=arm64-efi --efi-directory=/boot --bootloader-id=Gentoo  
grub-mkconfig -o /boot/grub/grub.cfg  

## Option 2 - Install bootloader (Systemd)  

mkdir -p /etc/portage/package.use   
echo "sys-apps/systemd-utils boot kernel-install" >> /etc/portage/package.use/systemd-utils   
emerge --oneshot --verbose sys-apps/systemd-utils   

bootctl install

sed -i 's/#timeout 3/timeout 10/g' /boot/loader/loader.conf  
echo default gentoo.conf >> /boot/loader/loader.conf  

echo -e "title Gentoo Linux" > /boot/loader/entries/gentoo.conf  
echo  -e "linux /$( ls -1 /boot/vm* | sed s/^.*\\\\/\\\//)" >> /boot/loader/entries/gentoo.conf   
echo "options root=/dev/sda3"  >> /boot/loader/entries/gentoo.conf   

## EFIStub

efibootmgr --create --disk /dev/sda --part 1 --label "Gentoo Linux EFISTUB" --loader /vmlinux-6.6.47-gentoo --unicode 'root='$(blkid | grep sda3 | awk '{print $5}' | sed 's/"//g')' rootfstype=ext4 rw initrd=\initramfs-6.6.47-gentoo.img'  

## Install linux mail

emerge acct-user/mail  

## Reboot system

exit  
cd  
umount -R /mnt/gentoo  
reboot  

## Configure Init Ramdisk (Only if required)

emerge sys-kernel/dracut  
dracut  
grub-mkconfig -o /boot/grub/grub.cfg

## Fix clock skew (only if alerts at startup)
cd /  
touch fixtime  
find . -cnewer /fixtime -exec touch {} \;   <-- ignore errors

