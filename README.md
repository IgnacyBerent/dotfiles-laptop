# Arch Linux Laptop Setup

<!--toc:start-->

- [Arch Linux Laptop Setup](#arch-linux-laptop-setup)
  - [Packages management](#packages-management)
    - [Save downloaded packages](#save-downloaded-packages)
    - [Pacman](#pacman)
    - [YAY](#yay)
  - [Symlinks](#symlinks)
    - [setting up fish](#setting-up-fish)
  - [Themes](#themes)
    - [Tmux plugins](#tmux-plugins)
    - [Gtk themes](#gtk-themes)
    - [Jupyter Lab](#jupyter-lab)
    - [Thunar](#thunar)
  - [Environment](#environment)
  - [Scripts](#scripts)
  - [Zen Browser](#zen-browser)
  - [Dotfiles Git](#dotfiles-git)
  - [Utils](#utils)
    - [NetworkManager](#networkmanager)
    - [SSD Maintainance](#ssd-maintainance)
    - [Pacman Cache Cleanup](#pacman-cache-cleanup)
    - [Btrfs Health Scrubbing](#btrfs-health-scrubbing)
    - [Battery & Hardware Optimizations](#battery-hardware-optimizations)
    - [Docker](#docker)
    - [Talscale](#talscale)
    - [Antivirus db](#antivirus-db)
  - [EDUROAM](#eduroam)
  - [Arch Linux install step by step (with encryption, btrfs, systemd boot)](#arch-linux-install-step-by-step-with-encryption-btrfs-systemd-boot)
    - [Partitions](#partitions)
    - [Set up LUKS](#set-up-luks)
    - [Create btrfs subvolumes](#create-btrfs-subvolumes)
    - [Mounting](#mounting)
    - [Pacstrap](#pacstrap)
    - [configs](#configs)
    - [mkinitcpio](#mkinitcpio)
    - [bootctl](#bootctl)
    - [Exit](#exit)
    - [Upgrade to CachyOS](#upgrade-to-cachyos)

<!--toc:end-->

---

## Packages management

### Save downloaded packages

```fish
  pacman -Qqen > pkglist.txt
  pacman -Qqem > pkglist-aur.txt

```

or:

```fish
  save_pkglist

```

### Pacman

```bash
sudo pacman -S --needed - < pkglist.txt
```

### YAY

```bash
git clone https://aur.archlinux.org/yay.git /tmp/yay
cd /tmp/yay
makepkg -si --noconfirm
cd ~
rm -rf /tmp/yay
yay -S --needed - < pkglist-aur.txt
```

## Symlinks

```bash
# avoid symlinking whole .config/ dir first by creating it on disk first
mkdir -p ~/.config
stow .
```

### setting up fish

```bash
chsh -s (which fish)
reboot
```

## Themes

### Tmux plugins

```bash
mkdir -p ~/.tmux/plugins/tpm && git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
~/.tmux/plugins/tpm/bin/install_plugins
systemctl --user enable tmux
```

### Gtk themes

```bash
papirus-folders -C cat-mocha-mauve --theme Papirus-Dark
gsettings set org.gnome.desktop.interface icon-theme "Papirus-Dark"
gsettings set org.gnome.desktop.interface gtk-theme "catppuccin-mocha-mauve-standard+default"
```

### Jupyter Lab

    settings -> theme -> Catppuccin Mocha

### Thunar

    Go to Thunar > Edit > Configure custom actions... > Open Terminal Here -> settings
    ghostty --working-directory=%f

## Environment

```bash
sudo vim /etc/environment
```

(paste)

```text
QT_QPA_PLATFORM="wayland;xcb"
ANKI_WAYLAND=1
MOZ_ENABLE_WAYLAND=1
```

## Scripts

```bash
chmod +x ~/.config/hypr/scripts/battery-notify.sh
chmod +x ~/.config/hypr/scripts/suncycle.sh
chmod +x ~/.config/hypr/scripts/toggle-suncycle.sh
```

## Zen Browser

```bash
flatpak install flathub app.zen_browser.zen
```

## Dotfiles Git

```bash
ssh-keygen -t ed25519 -C "2gb02ignac@gmail.com"
eval (ssh-agent -c)
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
GitHub > Settings > SSH and GPG keys -> New SSH Key
git remote set-url origin git@github.com:IgnacyBerent/dotfiles.git
```

## Utils

### NetworkManager

```bash
# Set network backend to iwd
nvim /etc/NetworkManager/conf.d/wifi_backend.conf
```

```text
[device]
wifi.backend=iwd
```

    systemctl enable NetworkManager

### SSD Maintainance

    sudo systemctl enable --now fstrim.timer

### Pacman Cache Cleanup

    sudo systemctl enable --now paccache.timer

### Btrfs Health Scrubbing

    # Scrub the root filesystem (/) monthly
    sudo systemctl enable --now btrfs-scrub@-.timer

    # Scrub the /home filesystem monthly
    sudo systemctl enable --now btrfs-scrub@home.timer

### Battery & Hardware Optimizations

    sudo systemctl mask power-profiles-daemon.service
    sudo systemctl enable --now tlp.service
    sudo systemctl enable --now bluetooth.service

### Docker

    sudo systemctl start docker.service
    sudo systemctl enable docker.service
    sudo groupadd docker
    sudo usermod -aG docker $USER
    newgrp docker

### Talscale

    sudo systemctl enable --now tailscaled.service

### Antivirus db

    sudo systemctl enable --now clamav-freshclam.service

## EDUROAM

```bash
sudo nmcli connection add type wifi con-name eduroam ifname wlan0 ssid eduroam \
wifi-sec.key-mgmt wpa-eap \
802-1x.eap ttls \
802-1x.phase2-auth pap \
802-1x.anonymous-identity "anonymous@your_uni_domain.pl" \
802-1x.identity "your_actual_username@your_uni_domain.pl" \
802-1x.password "your_password"
```

## Arch Linux install step by step (with encryption, btrfs, systemd boot)

### Partitions

```bash
# whipe everything
sgdisk -Z /dev/nvme0n1

# efi
sgdisk -n 1:0:+512M -t 1:ef00 -c 1:"EFI" /dev/nvme0n1

# root
sgdisk -n 2:0:0 -t 2:8309 -c 2:"LUKS" /dev/nvme0n1
```

### Set up LUKS

```bash
# Format the EFI partition
mkfs.fat -F 32 /dev/nvme0n1p1

# Initialize LUKS encryption (you will be prompted to create a password)
cryptsetup luksFormat /dev/nvme0n1p2

# Open the encrypted container and map it as 'cryptroot'
cryptsetup open /dev/nvme0n1p2 cryptroot
```

### Create btrfs subvolumes

```bash
mkfs.btrfs /dev/mapper/cryptroot

# Mount temporarily to create the subvolumes
mount /dev/mapper/cryptroot /mnt

# Create subvolumes
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@snapshots

# Unmount the top-level volume
umount /mnt
```

### Mounting

```bash
# Mount root subvolume
mount -o noatime,compress=zstd:1,space_cache=v2,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,boot}

# Mount home
mount -o noatime,compress=zstd:1,space_cache=v2,subvol=@home /dev/mapper/cryptroot /mnt/home

# Mount boot
mount /dev/nvme0n1p1 /mnt/boot

```

### Pacstrap

```bash
# make sure you are connected to the internet
ping -c 3 archlinux.org

# if not
iwctl
```

```bash
pacstrap -K /mnt base base-devel linux linux-headers linux-firmware intel-ucode btrfs-progs cryptsetup networkmanager iwd efibootmgr dosfstools nvim sudo git

genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

### configs

```bash
# Timezone (Poland)
ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc

# Enable both US English and Polish locales
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "pl_PL.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen

# Set system language (Defaults to English, but supports Polish characters)
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set Polish Programmer Keyboard for the console
echo "KEYMAP=pl" > /etc/vconsole.conf

# Hostname
echo "hostname" > /etc/hostname

# Root and user
passwd

useradd -mG wheel username
passwd username

EDITOR=nvim visudo # Uncomment %wheel line
```

### mkinitcpio

```bash
nvim /etc/mkinitcpio.conf

# set these hooks
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt btrfs filesystems fsck)

# rebuild
mkinitcpio -P
```

### bootctl

```bash
echo -e "default arch.conf\ntimeout 3\neditor no" > /boot/loader/loader.conf

# copy kernel image files
ls /boot | tee /boot/loader/entries/arch.conf
# copy UUID to arch.conf
blkid /dev/nvme0n1p2 | tee -a /boot/loader/entries/arch.conf
nvim /boot/loader/entries/arch.conf
```

Final form:

```txt
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options cryptdevice=UUID=copied-uuid:cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ rw
```

```bash
#make sure it works
bootctl list
```

### Exit

```
exit
unmount -R /mnt
reboot
```

### Upgrade to CachyOS

```bash
cd /tmp
curl https://mirror.cachyos.org/cachyos-repo.tar.xz -o cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz && cd cachyos-repo
sudo ./cachyos-repo.sh
sudo pacman -S linux-cachyos linux-cachyos-headers
sudo nano /boot/loader/entries/arch.conf
```

change these only:

```text
linux   /vmlinuz-linux-cachyos
initrd  /initramfs-linux-cachyos.img
```

```bash
# restart to change kernel
reboot

# remove old kernel
sudo pacman -Rns linux linux-headers
# make sure /boot is clear
ls /boot
# if not then try again
sudo pacman -Rns linux

```
