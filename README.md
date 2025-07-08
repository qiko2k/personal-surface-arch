# 🐧 arch linux install - full guide (btrfs + luks + secure boot)

> 👾 encrypted, snapshot-ready, clean  
> 💿 btrfs + luks + grub  
> 🔒 secure boot + sbctl  
> 🧠 tips + comments included  
> 💡 replace `youruser`, `yourhostname`, etc.

---

## ✨ credits

i would like to start this guide with a huge thank you to  
👉 **the rad lectures** — [youtube.com/@theradlectures](https://www.youtube.com/@theradlectures)  
this is almost a one-to-one transcription of his setup  
with some small tweaks to work better on the **microsoft surface laptop**.  
ty goat 🐐

---

## ☁️ connect to wifi (live usb)

```bash
passwd  # set temp root pw (needed for iwctl sometimes)

iwctl
station list  # shows wifi devices (e.g. wlan0)
station <interface> get-networks
station <interface> connect <your-ssid>  # enter wifi pw
exit

ping -c 2 google.com  # make sure it works
```

---

## 🕰 set time + sync

```bash
timedatectl set-timezone Europe/Stockholm  # or your tz
timedatectl set-ntp true  # enable ntp sync
```

---

## 💿 wipe + partition

```bash
lsblk  # find your disk (e.g. /dev/nvme0n1)

gdisk /dev/nvme0n1
x      # expert mode
z      # zap gpt/mbt (wipe all partitions)
y
y

# now re-create partition table

gdisk /dev/nvme0n1
n      # new partition (efi)
<enter> <enter> +1G
ef00   # efi system

n      # new partition (root luks)
<enter> <enter> <enter> <enter>  # full disk
p      # print partition table
w      # write and save
y
```

---

## 🔐 luks + btrfs

```bash
cryptsetup luksFormat /dev/nvme0n1p2  # encrypt root
YES  # type it uppercase

cryptsetup luksOpen /dev/nvme0n1p2 main  # unlock luks as /dev/mapper/main

mkfs.btrfs /dev/mapper/main
mount /dev/mapper/main /mnt

# create btrfs subvolumes for root + home
cd /mnt
btrfs subvolume create @
btrfs subvolume create @home
cd -
umount /mnt

# mount root subvol first
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/main /mnt

mkdir /mnt/home

# mount home subvol
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/main /mnt/home

# setup efi partition
mkfs.fat -F32 /dev/nvme0n1p1
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

---

## 📦 install base system

```bash
pacstrap /mnt base
genfstab -U -p /mnt >> /mnt/etc/fstab  # generate mounts
arch-chroot /mnt  # chroot into the system
```

---

## 🌍 locale + hostname

```bash
ln -sf /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
hwclock --systohc  # set hardware clock

pacman -S neovim  # for editing config
nvim /etc/locale.gen  # uncomment en_US.UTF-8
locale-gen

echo "lang=en_US.UTF-8" >> /etc/locale.conf
echo "yourhostname" >> /etc/hostname
```

---

## 👤 user + sudo

```bash
passwd  # set root pw

useradd -m -g users -G wheel youruser
passwd youruser  # set user pw

mkdir -m 755 /etc/sudoers.d
echo "youruser ALL=(ALL) ALL" >> /etc/sudoers.d/youruser
chmod 0440 /etc/sudoers.d/youruser  # secure perms
```

---

## 🛠 packages + mirrors

```bash
pacman -S sudo reflector rsync

# update mirrorlist with fastest sweden servers
reflector -c Sweden -a 12 --sort rate --save /etc/pacman.d/mirrorlist

# install full base system + tools
pacman -Syu base-devel linux linux-headers linux-firmware \
  btrfs-progs grub efibootmgr mtools networkmanager \
  network-manager-applet openssh git iptables-nft ipset \
  firewalld acpid grub-btrfs

# install microcode based on your cpu
pacman -S amd-ucode  # or intel-ucode
```

---

## 🧬 initramfs

```bash
nvim /etc/mkinitcpio.conf

# add modules to MODULES=
MODULES=(pinctrl_amd surface_hid surface_aggregator surface_aggregator_registry surface_aggregator_hub surface_hid_core 8250_dw atkbd btrfs)

# add encrypt hook before filesystems
HOOKS=(base udev autodetect modconf block encrypt filesystems keyboard fsck)

mkinitcpio -p linux
```

---

## 🧾 install grub

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

---

## 🧬 enable luks decryption in grub

```bash
blkid  # get uuid of luks partition (nvme0n1p2)
nvim /etc/default/grub

# set this:
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_iommu=off iommu=off cryptdevice=UUID=xxxx-xxxx:main root=/dev/mapper/main acpi_rev_override=1"

grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 📡 enable services

```bash
systemctl enable NetworkManager
systemctl enable sshd
systemctl enable firewalld
systemctl enable reflector.timer
systemctl enable acpid

exit
reboot
```

---

## 🛜 wifi + yay (aur)

```bash
nmcli device wifi connect <ssid> --ask  # connect to wifi
ping -c 2 google.com  # confirm net

git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si  # install yay
```

---

## ⏳ timeshift + autosnap

```bash
yay -S timeshift timeshift-autosnap

# show snapshot targets
sudo timeshift --list-devices

# create first snapshot
sudo timeshift --create --comments "[july 7 2025] start of time" --tags d
```

---

## ⚙️ editor + grub snapshot support

```bash
sudo su
cd ~
echo "export EDITOR=nvim" > .bashrc
source .bashrc

# grub-btrfs shows snapshots in boot menu
systemctl edit --full grub-btrfsd

# change snapshot path to match timeshift
# remove /.snapshots and use:
#   -t /timeshift/snapshots

sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 🧠 zram swap (compressed ram swap)

```bash
sudo pacman -S zram-generator
mkdir -p /etc/systemd/zram-generator.conf.d/
nvim /etc/systemd/zram-generator.conf.d/zram.conf
```

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
```

```bash
systemctl daemon-reexec
systemctl start /dev/zram0
```

---

## 🔐 secure boot (optional but powerful)

```bash
sudo pacman -Syu sbctl
sudo sbctl create-keys
sudo sbctl status
sudo sbctl enroll-keys --microsoft  # microsoft tag is optional, read more about it on arch wiki

# sign kernel + grub
sudo sbctl sign -s /boot/vmlinuz-linux
sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --disable-shim-lock --modules="tpm" --recheck
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo sbctl sign -s /boot/efi/grub/grubx64.efi
```

---

## 🔥 ufw firewall (optional)

```bash
sudo pacman -S ufw
sudo ufw enable
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
```

---

## 🩹 optional fix if you need it (surface laptop suspend issue)

> 🛠️ without this, suspend/resume will probably break  

---

### ⚡ override acpi

1. edit your grub defaults:  
   ```bash
   sudo nvim /etc/default/grub
   ```
2. update the `GRUB_CMDLINE_LINUX_DEFAULT` line to include `acpi_rev_override=1` **at the end**, for example:

   ```diff
   -GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_iommu=off iommu=off cryptdevice=UUID=xxxx-xxxx:main root=/dev/mapper/main"
   +GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet amd_iommu=off iommu=off cryptdevice=UUID=xxxx-xxxx:main root=/dev/mapper/main acpi_rev_override=1"
   ```

   > • replace `xxxx-xxxx` with the actual PARTUUID of `/dev/nvme0n1p2`  
   > • keep all other parameters in place  

3. regenerate your grub config:  
   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```
4. reboot and test suspend/resume – should now work 💤

---

## 🏁 you're done
> reboot into your encrypted arch install  
> keep your AUR usage low — more stability, fewer surprises  
> use `timeshift` often (before big changes especially)  
> consider setting a bios admin password + locking down grub  
> stay safe 🫡
