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
2 &emsp; swap &emsp;1GB  
3 &emsp; ext2 &emsp; 62GB  


mkfs.vfat -F 32 /dev/sda1         
mkfs.ext4 /dev/sda3

mkswap /dev/sda2      
swapon /dev/sda2

mount /dev/sda3 /mnt/gentoo

## Download Stage 3

cd /mnt/gentoo

links gentoo.org     
choose arm stage3 openrc     

tar xpvf st\* --xattrs-include='\*.\*' --numeric-owner

nano /etc/portage/make.conf    

MAKEOPTS="-j2"  
ACCEPT_LICENSE="-* @FREE @BINARY-REDISTRIBUTABLE"  
GRUB_PLATFORMS="efi-64"  
CPU_FLAGS_ARM="edsp neon neon-fp16 thumb vfp vfpv3 vfpv4 vfp-d32 aes sha1 sha2 crc32 asimd asimddp asimdfhm asimdhp v4 v5 v6 v7 v8 thumb2"  


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

ln -sf /usr/share/zoneinfo/America/Vancouver /etc/localtime


sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen  
locale-gen

eselect locale list < get locale
eselect locale set 

echo "LANG=en_US.UTF-8" >> /etc/locale.conf

## Configure Kernel

echo "sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE" | tee -a /etc/portage/package.license

emerge sys-kernel/linux-firmware  
emerge sys-kernel/gentoo-sources   
emerge net-wireless/wireless-regdb

eselect kernel list  
eselect kernel set 1

cd /usr/src/linux  
make menuconfig <-- save file

do Parameters below See below for GUI and Sound

make -j2 Image  
make install  
make modules  
make modules_install

sed -i ‘s/^# CONFIG_DRM is not set/CONFIG_DRM=y/’ .config  
sed -i ‘s/^# CONFIG_DRM_KMS_HELPER is not set/CONFIG_DRM_KMS_HELPER=y/’ .config  
sed -i ‘s/^# CONFIG_DRM_SIMPLEDRM is not set/CONFIG_DRM_SIMPLEDRM=y/’ .config  
sed -i ‘s/^# CONFIG_DRM_VIRTIO_GPU is not set/CONFIG_DRM_VIRTIO_GPU=y/’ .config  
sed -i ‘s/^# CONFIG_FRAMEBUFFER_CONSOLE is not set/CONFIG_FRAMEBUFFER_CONSOLE=y/’ .config  
sed -i ‘s/^# CONFIG_INPUT_EVDEV is not set/CONFIG_INPUT_EVDEV=y/’ .config  
sed -i ‘s/^# CONFIG_DEVTMPFS is not set/CONFIG_DEVTMPFS=y/’ .config  
sed -i ‘s/^# CONFIG_DEVTMPFS_MOUNT is not set/CONFIG_DEVTMPFS_MOUNT=y/’ .config  
sed -i ‘s/^# CONFIG_SOUND is not set/CONFIG_SOUND=y/’ .config  
sed -i ‘s/^# CONFIG_SND is not set/CONFIG_SND=y/’ .config  
sed -i ‘s/^# CONFIG_SND_PCM is not set/CONFIG_SND_PCM=y/’ .config  
sed -i ‘s/^# CONFIG_SND_TIMER is not set/CONFIG_SND_TIMER=y/’ .config  
sed -i ‘s/^# CONFIG_SND_HDA is not set/CONFIG_SND_HDA=y/’ .config  
sed -i ‘s/^# CONFIG_SND_HDA_CORE is not set/CONFIG_SND_HDA_CORE=y/’ .config  
sed -i ‘s/^# CONFIG_SND_HDA_INTEL is not set/CONFIG_SND_HDA_INTEL=y/’ .config  
sed -i ‘s/^# CONFIG_SND_HDA_GENERIC is not set/CONFIG_SND_HDA_GENERIC=y/’ .config  
sed -i ‘s/^# CONFIG_SND_HDA_CODEC_ANALOG is not set/CONFIG_SND_HDA_CODEC_ANALOG=y/’ .config  

make olddefconfig


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

emerge sys-boot/efibootmgr
mkdir -p /etc/portage/package.use   
echo "sys-apps/systemd-utils boot kernel-install" >> /etc/portage/package.use/systemd-utils   
emerge --oneshot --verbose sys-apps/systemd-utils   

bootctl install

sed -i 's/#timeout 3/timeout 10/g' /boot/loader/loader.conf  
echo default gentoo.conf >> /boot/loader/loader.conf  

echo -e "title Gentoo Linux" > /boot/loader/entries/gentoo.conf  
echo  -e "linux /$( ls -1 /boot/vm* | sed s/^.*\\\\/\\\//)" >> /boot/loader/entries/gentoo.conf   
echo "options root=/dev/sda3"  >> /boot/loader/entries/gentoo.conf   

## Option 3 - EFIStub

emerge sys-boot/efibootmgr  
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

## Sound
/sys/module/snd_hda_intel/parameters/bdl_pos_adjemerge media-libs/alsa-lib  
emerge --ask media-sound/alsa-utils  
alsamixer  
edit first t lines in /sys/module/snd_hda_intel/parameters/bdl_pos_adj  -1, 64  
echo "options snd-hda-intel bdl_pos_adj=-1,64" > /etc/modprobe.d/snd-hda-intel.conf  
test sound with speaker-test -t wav -c 2  

