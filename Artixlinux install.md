# Artixlinux 

# Root privilege

Run as administrator 
• sudo su or sudo -i

## Make partition

Divide some part of storage into a pieces of partition

• cfdisk /dev/nvme0n1
/dev/nvme0n1p1 (1GB) type= "EFI system"
/dev/nvme0n1p2  (Rest of storage) type="Linux filesystem"
• lsblk

--Encryption--

For security purposes 

• cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
• cryptsetup open /dev/nvme0n1p2 artixcrypt

--Formatting filesystem--

Format the filesystem for boot partition and root partition

• mkfs.vfat -n ARTIXEFI -F32 /dev/nvme0n1p1
• mkfs.btrfs /dev/mapper/artixcrypt

--Mounts--

Mounting all files

• mount /dev/mapper/artixcrypt /mnt
• btrfs subvolume create /mnt/@
• btrfs subvolume create /mnt/@home
• btrfs subvolume create /mnt/@.snapshots
• btrfs subvolume create /mnt/@var_log
• btrfs subvolume create /mnt/@var_cache
• btrfs subvolume create /mnt/@swap
• umount -l /mnt 
•  MO=rw,nodev,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,commit=150
• mount -o $MO,subvol=/@ /dev/mapper/artixcrypt /mnt
• mkdir -p /mnt/{home,.snapshots,var/{cache,log},swap,efi}
• mount -o $MO,subvol=/@home /dev/mapper/artixcrypt /mnt/home
• mount -o $MO,subvol=/@.snapshots /dev/mapper/artixcrypt /mnt/.snapshots
• mount -o $MO,subvol=/@var_cache /dev/mapper/artixcrypt /mnt/var/cache
• mount -o $MO,subvol=/@var_log /dev/mapper/artixcrypt /mnt/var/log
• mount -o rw,nodev,noatime,nodatacow,nodatasum,ssd,discard=async,subvol=/@swap /dev/mapper/artixcrypt /mnt/swap
• mount -t vfat /dev/nvme0n1p1 /mnt/efi

--Pacman settings--

Make a download faster and easter egg

• nano /etc/pacman.conf
#ParallelDownloads=5  --remove (#) and change 5 to 10
#Color  --remove (#)
ILoveCandy --add this at the first line of  #Misc options section

--Base system--

Install a basic packages system

• basestrap /mnt base base-devel linux-zen linux-firmware intel-ucode neovim btrfs-progs
• sudo cp /etc/pacman.conf /mnt/etc/pacman.conf

--Fstabgen--

Auto detect mounting filesystem

• fstabgen -U /mnt >> /mnt/etc/fstab
• nano /mnt/etc/fstab  --check the result if there is an error or modify something

--Chroot--

Change root from iso to installed system

• artix-chroot /mnt

--Swap--

Taking some storage for virtual ram

• btrfs filesystem mkswapfile --size 8g /swap/swapfile
• swapon /swap/swapfile --fstab settings needed
• echo "/swap/swapfile    none    swap    defaults    0    0"  >> /etc/fstab

--Packages--

Install some necessary packages

• pacman-key --init
• pacman-key --populate artix
• pacman -Sy artix-keyring
• pacman -S dinit elogind elogind-dinit mkinitcpio egummiboot efibootmgr efivar cryptsetup iwd iwd-dinit dbus dbus-dinit udev pipewire pipewire-dinit wireplumber git bash alacritty fastfetch man-db sbctl sudo snapper snap-pac btrfs-assistant wget curl yazi less which zstd cryptsetup-dinit polkit openresolv dosfstools mtools noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra ttf-dejavu ttf-liberation ttf-jetbrains-mono-nerd niri wayland xwayland-satellite ufw ufw-dinit mesa intel-media-driver vulkan-intel greetd greetd-dinit greetd-agreety

--Services--

Activate some necessary services

• ln -s /etc/dinit.d/elogind /etc/dinit.d/boot.d/
• ln -s /etc/dinit.d/udev /etc/dinit.d/boot.d/
• ln -s /etc/dinit.d/cryptsetup /etc/dinit.d/boot.d/
• ln -s /etc/dinit.d/dbus /etc/dinit.d/boot.d/
• ln -s /etc/dinit.d/iwd /etc/dinit.d/boot.d/
• ln -s /etc/dinit.d/ufw /etc/dinit.d/boot.d/
• ln -s /etc/dinit.d/greetd /etc/dinit.d/boot.d/

--Time--

Make a timezone

• ln -sf /usr/share/zoneinfo/Asia/Singapore /etc/localtime
• hwclock --systohc

--Localization--

• export LANG=en_US.UTF-8
• export LC_COLLATE=C
• echo LANG=$LANG > /etc/locale.conf
• echo KEYMAP=us > /etc/vconsole.conf
• nano /etc/locale.gen --remove (#) #en_US.UTF-8
• locale-gen

--User settings--

Make a username and add user to sudo privilege

• echo artix-asus > /etc/hostname
• useradd -m -G wheel -s /bin/bash maxwellbtw
• passwd maxwellbtw
• passwd
• EDITOR=nano visudo --remove (#) #%wheel ALL=(ALL:ALL) ALL

--Networks--

• nano /etc/hosts
   127.0.0.1        localhost
   ::1                   localhost
   127.0.1.1        artix-asus.localdomain  artix-asus

--UKI--

Use mkinitcpio for generate a kernel and make a uki

• blkid /dev/nvme0n1p2
• nano /etc/kernel/cmdline
cryptdevice=UUID=numberluksid:artixcrypt root=/dev/mapper/artixcrypt rootfstype=btrfs rootflags=subvol=/@ rw
• sudo cp /etc/kernel/cmdline /etc/kernel/cmdline_fallback
• nano /etc/mkinitcpio.conf
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
• nano /etc/mkinitcpio.d/linux-zen.preset
# mkinitcpio preset file for the 'linux-zen' package
#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-zen"
PRESETS=('default' 'fallback')
#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-zen.img"
default_uki="/efi/EFI/Linux/artix-linux-zen.efi"
#default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-zen-fallback.img"
fallback_uki="/efi/EFI/Linux/artix-linux-zen-fallback.efi"
fallback_options="-S autodetect --cmdline /etc/kernel/cmdline_fallback"
• mkdir -p /efi/EFI/Linux
• mkinitcpio -p linux-zen

--Secure boot--

Make a Artixlinux support secure boot

• sbctl status
• sbctl create-keys
• sbctl enroll-keys --microsoft
• export ESP_PATH=/efi
• sbctl verify --list file needed to signed
• sbctl sign --save /efi/EFI/Linux/artix-linux-zen.efi
• sbctl sign --save /efi/EFI/Linux/artix-linux-zen-fallback.efi

--Boot entries--

Add the boot entries into UEFI

• efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "ArtixLinux-linux-zen" --loader 'EFI\Linux\artix-linux-zen.efi' --unicode
• efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "ArtixLinux-linux-fallback" --loader 'EFI\Linux\artix-linux-zen-fallback.efi' --unicode

--Reboot--

To make sure if hooks is work run pacman -S linux-zen 

• ls -la /efi/EFI/Linux/artix-linux-zen.efi
• sbctl verify --make sure the list file is signed
• exit
• swapoff -a
• umount -R /mnt
• cryptsetup close artixcrypt
• reboot


References:
https://wiki.archlinux.org/title/User:Bai-Chiang/Arch_Linux_installation_with_unified_kernel_image_(UKI),_full_disk_encryption,_secure_boot,_btrfs_snapshots,_and_common_setups
https://codeberg.org/sabuj/Artix-Install.git#1
https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html
https://gist.github.com/suconakh/6f01784c2eb641d7ae20d5c12c6fc51d#file-artix_runit_btrfs_install-sh
https://wiki.artixlinux.org/Main/Installation
https://packages.artixlinux.org/
https://wiki.archlinux.org/title/Btrfs#Swap_file
https://wiki.archlinux.org/title/Unified_kernel_image
https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot
https://btrfs.readthedocs.io/en/latest/




