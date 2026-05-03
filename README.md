# Switch-from-Windows-to-Arch-Linux

## Overview

Switching from Windows to Arch Linux. The goal is to gain deeper technical knowledge of system operations, achieve greater control over the operating system, and build a minimal, efficient system from the ground up.

### Why Arch Linux?

#### Advantages

- Packages are available to Arch users before most other distributions.
- Rolling release model no version numbers, each component updates independently.
- Access to the **Arch User Repository (AUR)**, one of the largest community driven package repositories.
- Light weight and minimal by design.

#### Disadvantages

- Highly technical and user dependent, requires solid technical knowledge to utilize the system's full potential.
- Not beginner friendly or user friendly.

### Setup

- A device compatible with Arch Linux
- A portable USB drive (for file backup and bootable media)
- [Arch Linux ISO](https://archlinux.org/)
- [USBImager](https://gitlab.com/bztsrc/usbimager)

---

### Steps

#### Things to check before installation

1. Verify hardware compatibility with Arch Linux.
2. Back up current HDD/SSD and transfer important files to a portable USB drive.
3. Note down all required passwords and account credentials.

---

#### Preparing the Installation Media

1. Download the latest Arch Linux ISO from https://archlinux.org/.
2. Download and install USBImager from https://gitlab.com/bztsrc/usbimager.
3. Flash the ISO image onto the USB drive using USBImager.
4. Connect the USB drive, reboot the device, and boot into the Arch Linux installation media.

---

#### Post Boot: Connecting to WiFi

Once booted into the Arch Linux live environment, establish a network connection.

```bash
ip addr show
```

Displays current IP address. Confirms internet connectivity if an IP is already assigned.
 
```bash
iwctl
```

Launches the `iwd` interactive prompt for managing wireless connections.

```bash
station wlan0 get-networks
```

Lists all available WiFi networks.
 
```bash
iwctl --passphrase "PASSWORD" station wlan0 connect WIFINAME
```

Connects to the specified WiFi network using the provided password.

---

#### I’ve chosen the manual installation method in this process

- Partitioning the Disks

```bash
lsblk -f
```

Lists all storage volumes attached to the device. I’ve got one HDD and one SSD attached to my device, and here’s how I’ve partitioned them for my usage:

- **SSD** is used for EFI + Boot + encrypted LVM
- **HDD** is used for Home directory

---

#### Partition the SSD

```bash
fdisk /dev/SSDNAME
```

Opens the disk partitioning tool for the SSD.


```bash
p
```

Displays the current partition table.

```bash
g
```

Creates a new empty GPT partition table. 

```bash
n
```

Creates the first partition — **EFI partition**

- Partition number: default
- First sector: default
- Last sector: `+512M`

```bash
t
```

Sets the partition type.
  
- Partition number: `1`
- Partition type: `1` → `EFI System`

```bash
n
```

Creates the second partition — **Boot partition**
 
- Partition number: default
- First sector: default
- Last sector: `+2G`

```bash
n
```

Creates the third partition — **Linux system (LVM)**, using all remaining space.
 
- Partition number: default
- First sector: default
- Last sector: default

```bash
t
```

Sets the partition type for the third partition.
 
- Partition number: `3`
- Partition type: `44` → `Linux LVM`

```bash
w
```

Writes changes to disk. **This will completely wipe the SSD.** 

---

#### Partition the HDD

```bash
fdisk /dev/HDDNAME
```

Opens the disk partitioning tool for the HDD. 

```bash
g
```

Creates a new empty GPT partition table. 

```bash
n
```

Creates one partition using all available space — this will be used for `/home`.
 
- Partition number: default
- First sector: default
- Last sector: default

```bash
w
```

Writes changes to disk. **This will completely wipe the HDD.** 

---

- Formatting the Partitions

```bash
mkfs.fat -F32 /dev/SSDNAMEp1
```

Formats the first SSD partition as FAT32 for EFI use. 

```bash
mkfs.ext4 /dev/SSDNAMEp2
```

Formats the second SSD partition as ext4 for `/boot`.

```bash
cryptsetup luksFormat /dev/SSDNAMEp3
```

Encrypts the third SSD partition (Linux LVM) using LUKS encryption. 

```bash
cryptsetup open --type luks /dev/SSDNAMEp3 lvm
```

> Opens the encrypted partition and maps it for LVM use.
> 

```bash
pvcreate /dev/mapper/lvm
```

Creates a physical volume on the mapped LVM device. 

```bash
vgcreate volgroup0 /dev/mapper/lvm
```

Creates a volume group named `volgroup0`. 

```bash
lvcreate -L 30GB volgroup0 -n lv_root
```

Creates a 30GB logical volume for the root filesystem. 

```bash
vgdisplay
```

> Displays information about the volume groups. Verify `volgroup0` appears correctly.
> 

```bash
lvdisplay
```

Displays information about the logical volumes. Verify `lv_root` appears correctly. 

```bash
modprobe dm_mod
```

Inserts the device mapper kernel module. 

```bash
vgscan
```

Scans for all volume groups.

```bash
vgchange -ay
```

> Activates all volume groups.
> 

```bash
mkfs.ext4 /dev/volgroup0/lv_root
```

Formats the root logical volume as ext4. 

```bash
mkfs.ext4 /dev/HDDNAME1
```

Formats the HDD partition as ext4 for `/home`.

---

- Mounting the File systems

```bash
mount /dev/volgroup0/lv_root /mnt
```

Mounts the root logical volume. 

```bash
mkdir /mnt/boot
mount /dev/SSDNAMEp2 /mnt/boot
```

Creates and mounts the boot partition. 

```bash
mkdir /mnt/boot/EFI
mount /dev/SSDNAMEp1 /mnt/boot/EFI
```

Creates and mounts the EFI partition inside boot. 

```bash
mkdir /mnt/home
mount /dev/HDDNAMEp1 /mnt/home
```

Creates and mounts the HDD partition as `/home`. 

---

- Installing Base Packages

```bash
pacstrap -i /mnt base
```

Installs the base Arch Linux system into the mounted root filesystem. 

---

- Generating the fstab File

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```

Generates the filesystem table for automatic mounting on boot. 

```bash
cat /mnt/etc/fstab
```

Verifies the fstab file. Output should display **four entries**:
 
- `/` → root LV on SSD
- `/boot` → SSD p2
- `/boot/EFI` → SSD p1
- `/home` → HDD p1

---

- Chrooting into the Installed System

```bash
arch-chroot /mnt
```

Changes root into the newly installed Arch Linux environment to complete setup.

---

- Setting Up Users

```bash
passwd
```

Sets the root password. 

```bash
useradd -m -g users -G wheel NAME
```

Creates a new user and adds them to the `wheel` group for sudo access. 

```bash
passwd NAME
```

Sets the password for the newly created user. 

---

- Installing Additional System Packages

```bash
pacman -S base-devel dosfstools grub efibootmgr gnome gnome-tweaks lvm2 mtools nano networkmanager openssh os-prober git curl wget zip unzip vim man net-tools rsync dnsutils sudo
```

Installs essential system packages including the bootloader, desktop environment, and common utilities. 

---

- Enabling SSH

```bash
systemctl enable sshd
```

Enables the SSH daemon to start on boot. 

---

- Installing the Linux Kernel

```bash
pacman -S linux linux-headers linux-lts linux-lts-headers
```

> Installs the latest Linux kernel and the LTS kernel along with their headers.
> 

```bash
pacman -S linux-firmware
```

Installs firmware files required for various hardware components. 

---

- Installing GPU Drivers

```bash
lspci
```

Lists all PCI devices, useful for identifying the GPU. 

```bash
pacman -S mesa
```

Installs the Mesa graphics driver for Intel/AMD GPUs. 

```bash
pacman -S libva-mesa-driver
```

Installs the VA-API driver for hardware video acceleration on Mesa. 

---

- Generating the RAM Disk (initramfs)

```bash
nano /etc/mkinitcpio.conf
```

Edit the HOOKS line to include encryption and LVM support: 

```
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

```bash
mkinitcpio -p linux
mkinitcpio -p linux-lts
```

Generates the initial RAM disk for both the standard and LTS kernels. 

---

- Configuring the System

```bash
nano /etc/locale.gen
```

Uncomment `en_US.UTF-8 UTF-8` to set the system locale. 

```bash
locale-gen
```

> Generates the selected locale.
> 

```bash
nano /etc/default/grub
```

Edit the GRUB command line to include the encrypted device. Replace `DEVICENAME` with your actual SSD device name: 

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=/dev/SSDNAMEp3:lvm root=/dev/volgroup0/lv_root quiet"
```

The mapper name after the colon must match what you used in `cryptsetup open`, which was `lvm`. 

```bash
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```

Installs GRUB for UEFI systems. It will use `/boot/EFI` automatically since it is already mounted. 

```bash
cp /usr/share/locale/en\\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```

Copies the GRUB locale file for English language support. 

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Generates the GRUB configuration file. 

```bash
systemctl enable gdm
systemctl enable NetworkManager
```

Enables the GNOME Display Manager and NetworkManager to start on boot. 

```bash
exit
```

Exits the chroot environment. 

```bash
umount -R /mnt
```

Recursively unmounts all file systems mounted under `/mnt`. 

```bash
reboot
```

Reboots into the new Arch Linux installation. 

---

---

- Post-Installation Setup

#### Initial Checks

- Verify network connectivity after reboot.
- Set the system language to English in GNOME Settings.

#### System Update

```bash
sudo pacman -Syu
```

Performs a full system update. 

### Enabling Multilib Repository

```bash
sudo nano /etc/pacman.conf
```

Make the following changes:
 
- Uncomment the `[multilib]` section to enable 32 bit package support.
- Uncomment the `Color` option for colored terminal output.

```bash
sudo pacman -Syu
```

Updates the system again after enabling the multilib repository.

### Installing Microcode Updates

```bash
sudo pacman -S amd-ucode
```

Installs AMD CPU microcode updates. Replace with `intel-ucode` for Intel processors. 

#### Enabling Bash Completion

```bash
sudo pacman -S bash-completion
nano ~/.bashrc
```

Add the following line to `.bashrc`: 

```
complete -cf sudo
```

```bash
source ~/.bashrc
```

Reloads the `.bashrc` file to apply changes. 

#### Installing Common Applications

```bash
sudo pacman -S firefox libreoffice-fresh vlc
```

Installs Firefox browser, LibreOffice suite, and VLC media player. 

#### Installing YAY (AUR Helper)

```bash
sudo pacman -S git base-devel
git clone <https://aur.archlinux.org/yay.git>
cd yay
makepkg -si
```

Installs `yay`, a popular AUR helper for managing AUR packages. 

```bash
yay -S pacman-aur
```

Install the AUR extension. Go to preferences, enable AUR support and check for updates. 

#### Enabling SSD TRIM

```bash
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
systemctl status fstrim.timer
```

Enables and starts the periodic TRIM service for SSD health maintenance. 

#### Optimizing Mirrors with Reflector

```bash
sudo pacman -S reflector rsync
sudo reflector --latest 10 --sort rate --fastest 5 --save /etc/pacman.d/mirrorlist
```

Fetches and ranks the 10 most recently updated mirrors, selects the 5 fastest, and saves them as the active mirrorlist. 

#### Setting Up the Firewall (UFW)

```bash
sudo pacman -S ufw
sudo ufw enable
sudo ufw status verbose
sudo systemctl enable ufw.service
```

Installs, enables, and configures UFW (Uncomplicated Firewall) to start on boot. 

#### Setting Up System Snapshots with Timeshift

```bash
yay -Sy timeshift
```

Installs Timeshift for creating and managing system snapshots, useful for rollback in case of issues. 

---

### Results

- Successfully switched from Windows to Arch Linux.
- Configured full disk encryption using LUKS and LVM.
- Set up a fully functional GNOME desktop environment.
- Established AUR access, system snapshots, firewall rules, and optimized mirrors.

### Key Learnings

- In depth understanding of Linux disk partitioning, LVM, and LUKS encryption.
- How the Linux boot process works from initramfs to GRUB configuration.
- Managing packages with `pacman` and AUR tools like `yay`.
- Importance of system security practices such as disk encryption and firewall configuration.
- How rolling release distributions differ from versioned distributions.

### References

- [Arch Linux Official Website](https://archlinux.org/)
- [Arch Wiki](https://wiki.archlinux.org/)
- [USBImager](https://gitlab.com/bztsrc/usbimager)
- [YAY AUR Helper](https://github.com/Jguer/yay)
- [Timeshift GitHub](https://github.com/linuxmint/timeshift)
