# Arch Linux Laptop

Arch linux

## Connect to the internet
```shell
check the available network devices
# ip link

```
- https://wiki.archlinux.org/title/Network_configuration/Wireless#Rfkill_caveat
ensure the wifi-device isn't off, either by software or disabled through the "wifi-button"
```shell
rfkill list
```

- https://wiki.archlinux.org/title/Iwd#iwctl
```shell
iwctl # enters the interactive prompt
[iwd]# device list
[iwd]# station device scan
[iwd]# station device get-networks
[iwd]# station device connect SSID
# it works like magic, no output and it was connected. Exit and check the connection
ping 8.8.8.8
# as cli
iwctl --passphrase passphrase station device connect SSID
```

set a password for root
```shell
passwd
```

enable sshd, so we can continue the installation remotely, with copy and paste ;)
```shell
systemctl enable sshd
systemctl status sshd
ip a # check current ip
```

- adjust clock and timezone
```shell
date
timedatectl set-timezone TIMEZONE # there is autocomplete for TIMEZONE :)
date
timedatectl set-ntp true
date
ls /usr/share/zoneinfo # check available timezones if needed
```

- formatting the HD, the dangerous part!
list the devices, I find nice to use both, the complement each other
```shell
fdisk -l
lsblk
```
I had Windows 11 pre-installed, I resized it using Windows' Disk Management:
 - Right click on "Start menu", choose `Disk Management`
 - Right click on "C:", `Shirink Volume...`


```shell
# cgdisk is an ncurses-based GUID partition table manipulator
cgdisk /dev/nvme0n1
```
```
use    - sie  - filesystem - mount point - obs
------------------------------------------
boot   - 512M - ext4       - /boot
system - all  - ext4       - /           - encrypted
```

- encrypt the `/` 

```shell
cryptsetup -y --use-random luksFormat /dev/nvme0n1p8
cryptsetup luksOpen /dev/nvme0n1p8 mokonaroot
lsblk # check the new volume

# create logical volumes for / and swap
pvcreate /dev/mapper/mokonaroot
vgcreate mokona-vg /dev/mapper/mokonaroot
lvcreate -L 32G --alloc contiguous --name swap mokona-vg
```

- format the `boot` and the `/`
NO CONFIRMATION IS ASKED!
```shell
# format the boot partition
mkfs.ext4 /dev/nvme0n1p7

# format the cryptroot, a.k.a `/`
mkfs.ext4 /dev/mapper/mokona--vg-mokona
```

## Installing Arch Linux :)

### Prepare:

```shell
mount /dev/mokona-vg/mokona /mnt
mkdir /mnt/boot
mount /dev/nvme0n1p7 /mnt/boot
mkdir /mnt/boot/efi # mkdir /mnt/efi
# check which one is the EFI partition
fdisk -l
mount /dev/nvme0n1p1 /mnt/boot/efi
```

### Install

```shell
# `lvm2` is needed if using LVM on LUKS as I'm. If not present, the boot will decrypt the partition, but the volume group will be missing
pacstrap /mnt linux linux-firmware base base-devel grub efibootmgr vim git intel-ucode networkmanager openssh wget curl man-db man-pages lvm2

# I'm using the EFI/bios menu to select windows, so no need to add it to grub,
# therefore `os-prober` is not needed. `cryptsetup` to decrypt partitions, including BitLocker ones, finally `ntfs-3g` to mount NTFS
pacstrap /mnt cryptsetup ntfs-3g
```

### Configure

```shell
# Generate Fstab
genfstab -U /mnt >> /mnt/etc/fstab

# "Log into" the new system :)
arch-chroot /mnt

# Set the timezone
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime

# adjust clock
timedatectl set-ntp true

# Set the Hardware Clock
hwclock --systohc

# Generate locale:
# comment in en_GB.UTF-8 UTF-8 in /etc/locale.gen
vim /etc/locale.gen
locale-gen

# Set LANG variable
echo "LANG=en_GB.UTF-8" >> /etc/locale.conf

# Set hostname
echo "mokona" >> /etc/hostname

# Initial Ramdisk Configuration:
#  Edit HOOKS in /etc/mkinitcpio.conf (order matters)
#  MODULES: sdd `vmd` if RAID is ON in the BIOS, or disable RAID
#   MODULES=(vmd)
#  Add `encrypt` and move `keyboard` before `modconf`
#   HOOKS=(base udev autodetect keyboard modconf block encrypt filesystems fsck)
vim /etc/mkinitcpio.conf

# Creating a new initramfs with the `linux` preset
mkinitcpio -p linux 
```

 - Setup GRUB
```shell
# Find the root partition UUID, here 
blkid

vim /etc/default/grub:
# GRUB_CMDLINE_LINUX="cryptdevice=UUID=${UUID from root partition}:cryptroot root=/dev/mapper/cryptroot"
# make sure that the lvm module is preloaded
# GRUB_PRELOAD_MODULES="... lvm"
# comment in
# GRUB_ENABLE_CRYPTODISK=y


vim /etc/grub.d/40_custom
# Add grub menu item for Win 11 by editing /etc/grub.d/40_custom
# #!/bin/sh
# exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
# if [ "${grub_platform}" == "efi" ]; then
#   menuentry "Windows 11" {
#     insmod part_gpt
#     insmod fat
#     insmod search_fs_uuid
#     insmod chain
#     # use:
#     # after --set=root, add the EFI partition's UUID
#     # this can be found with either:
#     #
#     # a. blkid
#     # - or -
#     # b. grub-probe --target=fs_uuid /boot/efi/EFI/Microsoft/Boot/bootmgfw.efi
#     #
#     search --fs-uuid --set=root 64C2-28EA
#     chainloader /EFI/Microsoft/Boot/bootmgfw.efi
#   }
# fi

# Install GRUB
grub-install --efi-directory=/boot/efi

# add GRUB_DISABLE_OS_PROBER=false to etc/default/grub
# not doing it anymore
# vim etc/default/grub

# Generate the grub configuration.
# Not on Win 11 + BitLocker: on my Dell XPS as Win 11 requires safe boot enable to boot, even thought
# the GRUB entry works, Win 11 asks the partition password to decrypt it. Besides just enabling safe boot
# makes the boot to ignore GRUB and boot directly on Win 11, whereas having safe boot disabled boots
# thought GRUB and Arch Linux works as expected. 
grub-mkconfig -o /boot/grub/grub.cfg

```

- Add user

```shell
# Set root password
passwd

# Create ainsoph and set its password 
useradd -m -G wheel ainsoph
passwd ainsoph

# Edit sudores
#  ## Uncomment to allow members of group wheel to execute any command
#  %wheel ALL=(ALL) ALL
visudo
```

 - Networking

```shell
systemctl enable NetworkManager
systemctl enable sshd
```

 - Out and reboot

```shell
exit
umount -R /mnt
reboot
```

 - Log in :)

- Disable root ssh
```shell
# add `PermitRootLogin no`, probably below `#PermitRootLogin prohibit-password`
sudo vim /etc/ssh/sshd_config

# check battery capacity
cat /sys/class/power_supply/BAT0/capacity
```

```shell
# connect to wifi
nmtui-connect

# update the system
sudo pacman -Syu

# clone the repo
mkdir -p ~/devel/github.com/AndersonQ
cd ~/devel/github.com/AndersonQ
git clone https://github.com/AndersonQ/linux-laptop.git
```



 - Install Gnome and GDM
 
```shell
[ainsoph@mokona AndersonQ]$ sudo pacman -S gnome gdm
[sudo] password for ainsoph:
:: There are 56 members in group gnome:
:: Repository extra
1) baobab  2) cheese  3) eog  4) epiphany  5) evince  6) gdm  7) gnome-backgrounds  8) gnome-calculator  9) gnome-calendar  10) gnome-characters  11) gnome-clocks
12) gnome-color-manager  13) gnome-connections  14) gnome-console  15) gnome-contacts  16) gnome-control-center  17) gnome-disk-utility  18) gnome-font-viewer  19) gnome-keyring
20) gnome-logs  21) gnome-maps  22) gnome-menus  23) gnome-music  24) gnome-photos  25) gnome-remote-desktop  26) gnome-session  27) gnome-settings-daemon  28) gnome-shell
29) gnome-shell-extensions  30) gnome-software  31) gnome-system-monitor  32) gnome-text-editor  33) gnome-tour  34) gnome-user-docs  35) gnome-user-share  36) gnome-weather
37) grilo-plugins  38) gvfs  39) gvfs-afc  40) gvfs-goa  41) gvfs-google  42) gvfs-gphoto2  43) gvfs-mtp  44) gvfs-nfs  45) gvfs-smb  46) malcontent  47) nautilus  48) orca  49) rygel
50) simple-scan  51) sushi  52) totem  53) tracker3-miners  54) xdg-desktop-portal-gnome  55) xdg-user-dirs-gtk  56) yelp

Enter a selection (default=all):
resolving dependencies...
:: There are 2 providers available for jack:
:: Repository extra
1) jack2  2) pipewire-jack

Enter a number (default=1): 2
:: There are 2 providers available for pipewire-session-manager:
:: Repository extra
1) pipewire-media-session  2) wireplumber

Enter a number (default=1): 1
:: There are 2 providers available for emoji-font:
:: Repository extra
1) noto-fonts-emoji  2) ttf-joypixels

Enter a number (default=1): 1
```
- Enable GDM service

```shell
sudo systemctl enable gdm.service
```


```
cd linux-laptop
make install-base

# use zsh
chsh -s $(which zsh)

make configure-user
```

 - Disable root ssh
```shell
# add `PermitRootLogin no`, probably below `#PermitRootLogin prohibit-password`
sudo vim /etc/ssh/sshd_config

# check battery capacity
cat /sys/class/power_supply/BAT0/capacity
```

 - Fix Gnome settings

 ```shell
 dconf load -f / < dconf.bkp
 ```

- Add finferprint to sudo: https://wiki.archlinux.org/title/fprint

edit `/etc/pam.d/sudo`
add `auth            sufficient      pam_fprintd.so` as the first one:
``````
auth            sufficient      pam_fprintd.so
auth            include         system-auth
```


- Add and configure Gnome extensions

Install [chrome-gnome-shell and gnome-browser-connector](https://wiki.gnome.org/action/show/Projects/GnomeShellIntegration/Installation)
```
git clone https://aur.archlinux.org/gnome-browser-connector.git
cd gnome-browser-connector
makepkg -si
```

- Droidcam

```shell
yay -S droidcam lantern-bin
```

## Using KeepassXC to manage SSH keys
https://ferrario.me/using-keepassxc-to-manage-ssh-keys/


## TO install
https://github.com/romkatv/powerlevel10k#oh-my-zsh
