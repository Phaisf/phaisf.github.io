# phaisf.github.io
# 1 Pre-Installation 
Make snapshot once booted into actual VM
## 1.1 Acquire an Installation Package
Visit the Arch Linux [Download](https://archlinux.org/download/) page, and for this guide, scroll down to the *HTTP Direct Downloads* section and click on the [mirror](https://archlinux.org/download/#download-mirrors) page link (linked in case page cannot be found).

Any mirror download works, but for this guide, choose the one under the *United States* section and choose the [mit.edu](https://mirrors.mit.edu/archlinux/iso/2025.10.01/). Next, download the *archlinux-2025.10.01-x86_64.iso* ISO file.

## 1.2 Verify Signature
Before using, it is recommended to verify the image signature before using it, especially with an *HTTP mirror*. On the *Download* page, under *HTTP Direct Downloads* check the *ISO SHA256* signature and then check that same signature with the SHA256.txt file in the chosen mirror download link. These two signatures should match 

(As of 2025.10.22: 86bde4fa571579120190a0946d54cd2b3a8bf834ef5c95ed6f26fbb59104ea1d)

## 1.3 Prepare VM 
For this guide, we are using Oracle VirtualBox. 
1. Select *New* to create a new VM. Name the VM
2. Select the mirror ISO Image that was downloaded in step 1.1 (Note OS info should autofill, if not just select Arch Linux for OS Dist. and OS Ver.)
3. Under **Specify virtual hardware** set **Base Memory** to at least <font color="#00b0f0">2GB</font>, **Number of CPUs** can be left at 1
4. Under **Specify virtual hard disk** set **Disk Size** to at least <font color="#00b0f0">20GB</font>
5. In **Display** set video memory to 128 MB.
6. **Finish**
7. Select **Settings**
8. Under **System** and **Motherboard** check **UEFI** then save it.

## 1.4 Boot into VM + Set Console Keyboard Layout & Font
Select **Start**
Select first option to install Arch Linux x86 medium
We won't be changing any settings here, but these commands are for future reference.
Once booted in, setup keyboard layout. Default layout is US, but to list available layouts:
```bash
# localectl list-keymaps
```
To set keyboard layout, using German keyboard layout as example:
```bash
# loadkeys de-latin1
```
To set console fonts (located in `/usr/share/kbd/consolefonts/`):
```bash
# setfont ter-132b
```
## 1.5 Verify Boot Mode + Internet Check + Update System Clock
To verify boot mode, check UEFI bitness: (determines processor architecture of the bootloader program that firmware will load)
```bash
# cat /sys/firmware/efi/fw_platform_size
```
- In this guide, the command should return `64` meaning it is booted in UEFI mode and has a 64-biit x64 UEFI.
- If it returns `No such file or directory`, might have forgotten to enable UEFI in settings (I did this, refer back to 1.3 step 8)
To check if there is a connection, use ping
```bash
# ping ping.archlinux.org
```
- Should return `64 bytes from redirect.archlinux.org...`
To check if system clock is synchronized
```bash
# timedatectl
```

## 1.6 Partitioning Disks
Identify disk name by using *fdisk*.
```bash
# fdisk -l
```
Names could be `/dev/sda`, `/dev/nme0n1`, or `/dev/mmcblk0`.
In this case, `/dev/sda` is ours. Ignore results ending in `rom`, `loop`, or `airootfs`.
### Pre-Steps
Start `fdisk` utility for the partition `/dev/sda`
```bash
fdisk /dev/sda
```
`o` - create new empty DOS partition table 
`g`- create new empty GPT partition table
`p`- verify `Disklabel type: gpt` and there are no partitions
### Create Partitions
Begin creating the following partitions
- For booting in UEFI mode: an EFI system partition.
- One partition for `[SWAP]`
- One partition for the root directory `/`.

#### EFI System
`n` - Create new partition 
`1` - Choose partition number (`/dev/sda1`)
`[Enter]` - Accept default starting sector for First Sector
`+1G` - Add 1 GiB to the partition in the Last Sector
Should display `Created a new partition 1 of type 'Linux filesystem' and size of 1 GiB.`
`t` - Change partition type
`1` - Select partition 1
`1` - Changes partition type to EFI System, 1 is alias for it in GPT table list
`p` - Verify partition type is shown as `Type: EFI System`.

#### `[SWAP]` Directory
`n` - Create new partition
`2` - Choose partition number (/dev/sda2)
`[Enter]` - Accept default starting sector for First Sector
`+4G` -  Add 4 GiB to the partition in the Last Sector
Should display `Created a new partition 2 of type 'Linux filesystem' and size of 4 GiB.`
`t` - Change partition type
`2` - Select partition 2
`19` - Changes partition type to Linux Swap, 19 is the partition type code
`p` - Verify partition type is shown as `Type: Linux Swap`.

#### Root `/`
`n` - Create new partition
`3` - Choose partition number (/dev/sda2)
`[Enter]` - Accept default starting sector for First Sector
`[Enter]` -  Add rest of the partition in the Last Sector
Should display `Created a new partition 3 of type 'Linux filesystem' and of size ... GiB`. Displays rest of the disk size.
`t` - Change partition type
`3` - Select partition 3
`23` - Changes partition type to Linux root (x86-64), 23 is the partition type code.
p - Verify partition type is shown as `Type: Linux root (x86-64)`.


## 1.7 Formatting Partitions
Format the EFI, SWAP, and Root partitions using `mkfs`
To format EFI
```bash
# mkfs.fat -F 32 /dev/sda1
```
To format SWAP, use a slightly different command, `mkswap`
```bash
# mkswap /dev/sda2
```
Usually need to enable swap space after formatting, do so using `swapon`:
```bash
# swapon /dev/sda2
```
To format Root:
```bash
# mkfs.ext4 /dev/sda3
```
## 1.8 Mount file systems
After formatting, the partitions need to be mounted so the system installer can write files to them. SWAP will not need to be mounted as it was enabled with `swapon` from previous step.
##### Create Mount point
Create Mount Point for EFI --> Mount EFI partition
Create a subdirectory where EFI partition will be mounted to
```bash
# mkdir -p /mnt/boot/efi
```
Mount EFI partition:
```bash
# mount /dev/sda1 /mnt/boot/efi
```
##### Mount root `/`
Mount root to primary mount point which is usually `/mnt`
```bash
# mount /dev/sda3 /mnt
```
# 2 Installation
## 2.1 Installing
Install a few essential packages first
```bash
# pacstrap -K /mnt base linux linux-firmware
```
# 3 Configure the system
## 3.1 Fstab + Chroot
Generate Fstab file
```bash
# genfstab -U /mnt >> /mnt/etc/fstab
```
Change root into the new system to directly interact with the new system's environment
```bash
# arch-chroot /mnt
```
## 3.2 Install Desktop Environment (DE)
Because the guide does not provide a DE installation guide, we must download our own DE, in this case GNOME
`pacman -S` - Package manager for Arch Linux where the -S flag is used to Sync packages
Use `pacman` to install a DE
- Install base graphics first, Xorg in this case which is a display server which acts as a bridge between the DE and hardware. 
```bash
# pacman -S xorg
```
- Install the DE, in this case gnome with the `gnome-extra` flag to install extra popular applications.
```bash
# pacman -S gnome gnome-extra
```
- Set the system to boot into the GUI DE
```bash
# systemctl enable gdm.service
```
## 3.3 Time
Run this command to show the correct local time and handle Daylight Saving Time, set the time zone by:
```bash
# # ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```
Run hwclock to generate `/etc/adjtime` which ensures the hardware clock is set to UTC
```bash
# hwclock --systohc
```
## 3.4 Localization + Nano
To configure localization, use a text editor like `nano`
To install, sync packages first
```bash
# pacman -Sy
```
Then install `nano`
```bash
# pacman -S nano
```
Generate locales for different regions by using the command:
```bash
# locale-gen
```
After, using nano, check the file and ensure that the following line is uncommented
```bash
# sudo nano /etc/locale.gen 
# en_US.UTF-8 UTF-8
```
Set the language variable by creating and editing the file `/etc/locale.conf` and set the `LANG` variable.
```bash
# sudo nano /etc/locale.conf
# Add this line if not already present:
# LANG =en_US.UTF-8
```
Set the console keyboard layout to the standard U.S. QWERTY layout
Create and edit the `/etc/vconsole.conf`
```bash
# sudo nano /etc/vconsole.conf
```
Ensure the following line is in this file, if not add then save.
```bash
# KEYMAP=us
```
## 3.5 Network configuration
To assign an identifiable name to the system, create the hostname file
```bash
# sudo nano /etc/hostname
```
Add an identifiable name to the file, in this case `archlinux`.
Install a network manager by installing the package `networkmanager`.
```bash
# pacman -S networkmanager
```
Enable and start the service by using `systemctl`, if start is being ignored, that is okay move on and check on startup with `status` command.
```bash
# systemctl enable NetworkManager
# systemctl start NetworkManager
```
## 3.6 Root password
Set a secure password for the root user in this case we use `admin` as the password after running the following command.
```bash
# passwd
```
## 3.7 Boot loader
This part is pretty important, I originally rebooted my VM without knowing why it was still booting into the terminal part instead of the DE. It was because I didn't have a boot loader.
Choose and install a Linux-capable boot loader, `GRUB` here.
```bash
# pacman -S grub efibootmgr
```
Install GRUB to the Disk. I had to mount my EFI partition again for this command to work as it wasn't finding my EFI partition.
```bash
# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --removable
```
Generate a config file which will tell the bootloader where the kernel is and how to start the operating system.
```bash
# grub-mkconfig -o /boot/grub/grub.cfg
# -o flag writes results to file
```
After these have been successfully installed, exit and reboot
```bash
# exit
# reboot
```
## 3.8 Booting into environment
Shut down VM
Open settings
Go to Storage
Remove the ISO Under Storage by clicking on the ISO file and selecting the CD icon the right. 
Then select Remove Disk from Virtual Drive 
Start the VM back up and select the Root account to log into the DE
## 3.9 Shell + Terminal Customization + Adding users
Open up a new terminal to install these extra packages and do the following configs.
Add two users, a personal user and the `codi` user.
```bash
# sudo useradd -m -G wheel,users -s /bin/bash phillip
```
`-m` will create the user's home directory and `-G wheel,users` will add the user to the `wheel` group (for `sudo`) and the standard `users` group. 
`-s /bin/bash` will set the default shell to Bash.
Set a secure password for the user
```bash
# sudo passwd phillip
# wacom480 <-- in this case the password for phillip is what the arrow is pointing to.
```
Repeat the process for the `codi` user and set password as `admin`.
Install a different package manager, in this case zsh
```bash
# pacman -S zsh
```
Install `ssh`
```bash
# pacman -S openssh
```
The SSH server is managed by a system service called `sshd` (SSH Daemon) and it must be enabled to ensure it automatically starts upon reboot
```bash
# systemctl enable sshd.service
```
Add color coding to the terminal by setting environment variables in the user shell config file (`bashrc`). We use a text editor like `nano` to edit the config file.
```bash
# sudo nano ~/.bashrc
# Add the following line for the ls command
# alias ls='ls --color=auto'
```
This alias creates a shortcut where `ls` will be substituted for `ls --color=auto` where the flag tells `ls` to output colors when  it detects that output is going to a terminal.

Add some common aliases to create short commands for some frequently used ones. These will be added to the same `~/.bashrc` file.
```bash
# alias ll='ls -alF'
# alias c='clear'
```
`ll='ls -alF'` - Shortcuts `ls` with flags to show hidden files, in a detailed format, and show file type indicators.
`c='clear'` - Shortcuts `clear` to clear the terminal.

Lastly, install a browser, in this case we install FIrefox.
```bash
# sudo pacman -S firefox
```



