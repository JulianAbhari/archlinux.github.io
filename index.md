## Arch-Installation

Before you start I want to preface by saying that this is the first time I've ever installed arch so each command might not have that much info explaining why it's used. Another thing I can't stress enough is to take snapshots after each step and preferably after each sub-step, I can't tell you just how useful it was for me to immediately back up into an immediate snapshot.

1. After you’ve loaded in your iso into your vm-ware environment and configured your virtualization to have the correct amount of memory and storage, verify the boot mode by entering this command: `ls /sys/firmware/efi/efivars`
- If the directory exists then the system is booted in UEFI mode


2. Connecting to the internet:
- See if you’re already connected
- ping -c 2 google.com
- If you see that your packets were transmitted, and you received packets back from google, then that’s one way to make sure you’re connected.
- In my case this worked
- Another way to tell is by using curl
- curl -I https://linuxhint.com/
- If you get an http status of 200 then you know you’re for sure connected to the internet

3. Update the system clock
- Check if timedate needs updating
- `timedatectl status`
- If it does you can set the time using the following command
- `timedatectl set-time “2021-10-19 16:32:25”`
- To not continuously have to set-time each time you reboot and the time isn’t correct, se the time synchronization check to true using this command
- `timedatectl set-ntp true`

4. Partition the disks
- Type `fdisk -l` to list all available disks
- Type `fdisk /dev/sda` to create a partition in the correct disk
- Typing m brings up a help page which is pretty useful
- I typed `g` to create a gpt partition table
- Type `n` to add a new partition
   - Partition number: `1`
      - When asking which sector I simply pressed return to use the default value of 2048
      - When asking for the last sector I typed `+550M` as that will be the EFI partition
      - I’m adding two more partitions, so I type `n` to create another new partition
    - Partition number: `2`
      - Default sector
      - Size +2G because this next partition is for SWAP: `+2G`
      - Press `n` one last time to add the last partition
    - Partition number: `3`
      - When asking for first sector use default
      - When asking for last sector use default because that’s the remaining space available
- Now so far every partition has been of type “Linux filesystem” which is only correct for the third partition, but not for the first 2
- Press `t`  to change partition 1’s type to EFI System
- Press `1` to specify that we want to use the first type, EFI System
- Press `t` to change partition 2’s type
- Press `19` to specify that we want to use the 19th type, Linux swap
   - I accidentally typed 18, instead of 19, and I had to recover by pulling from my last snapshot. 
- Make sure everything is correct before this next step
- Type `w` to write the partition table
5. Make file system
- Make f3 different file systems on 3 different partitions
- For EFI partition it needs to be FAT32, so type `mkfs.fat -F32 /dev/sda1`
- For swap partition use `mkswap /dev/sda2`
- Turn swap on by typing `swapon /dev/sda2`
- Make filesystem on sda3, by using `mkfs.ext4 /dev/sda3`
- Now to mount the biggest partition, use `mount /dev/sda3 /mnt`

6. Installation
- To install the base system for arch run the command: `pacstrap /mnt base linux linux-firmware`
- Generate file system table: `genfstab -U /mnt >> /mnt/etc/fstab`
- Use `arch-chroot /mnt` to change into root directory of new installation
- Now that you’re in your new installation, the font should’ve changed
- Set time zone using `ln -sf /usr/share/zoneinfo/[region]/[city] /etc/localtime`
- I did `ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime`
- Set hardware clock using: `hwclock --systohc`
- Now we need to use nano to edit the local and uncomment our specific locale
- Before we can use nano though, we have to install its package
- Use `pacman -S nano`
- Type `nano /etc/locale.gen`
- In my case I uncommented the line `en_US.UTF-8 UTF-8`
- To get out of nano simply press control+x and then Y to save changes and then enter
- Now run this command: `locale-gen`
- Now we need to set the hostname using `nano /etc/hostname`
- Once in the hostname I just entered the hostname of the computer, `archvbox` and saved the file by pressing control+X and hitting Y to confirm the save
- Now to create the host file `nano /etc/hosts`
- Following the wiki simply add in these lines: 
`127.0.0.1        localhost`
`::1              localhost`
`127.0.1.1        [myhostname (in my case ‘archvbox’)]`

7. Creating users
- After saving the host file, we need to create users and passwords
- Create a root password by entering `passwd`
- Create a new user, in my case I’m going to add a user named ‘sal’
- Type command: `useradd -m sal`
- I’m setting a password for sal by typing `passwd sal`
- In order to have the password be force expiration, after setting user sal’s password, I’m typing `passwd -e` after setting the password
- I want to add sudo to the main user but to do that I need to install sudo, `pacman -S sudo`
- Edit the sudo file and add the users that have access to sudo using: `EDITOR=nano visudo`
- Uncomment the line that says `%wheel ALL=(ALL) ALL`
- Now, use this command to add sudo privileges to the user of your choice: `usermod -aG wheel [user]` (wheel is an alias for sudo)

8. Install grub: `pacman -S grub`
- Install efibootmgr: `pacman -S efibootmgr dosfstools os-prober mtools`
- Make an efi directory and boot directories: `mkdir /boot/EFI`
- `mount /dev/sda1 /boot/EFI`
- `grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck`
- Make grub installation file: `grub-mkconfig -o /boot/grub/grub.cfg`

9. Install network manager
- `pacman -S networkmanager vim` This uses pacman to install the networkmanager and vim

10. Install ssh
- `pacman -Syu` 
- `pacman -S openssh` These two pacman installs are used to download dependencies for ssh

11. Enable network manager
- `systemctl enable NetworkManager` This actually enables the network manager

12. Reboot
- Exit with `exit`
- Unmount current partition using `unmount -l /mnt`
- Reboot the VM to finish basic installation with `reboot`

13. Install LXDE
- Install a lightweight desktop envrionment`sudo pacman -S lxde`
- Install desktop environment dependencies `sudo pacman -S xorg-xinit`
- Ues nano to edit the .xinitrc file for the desktop environment `nano .xinitrc`
- Enter `exec startlxde` into the blank space
- Enter `startx` to load up the desktop environment

14. Installing git
- `sudo pacman -S git`

15. Installing browser
- `git clone https://aur.archlinux.org/google-chrome`
- `sudo pacman -S base-devel`
- `makepkg -s` 
- Find the proper compressed package that ends in .tar.gz through ‘ls’
- `sudo pacman -U google-chrome-95.0.4638.54-1-x86_64.pkg.tar.zst`
- I was wondering why the process for installing chrome took so many steps, but I found a YouTube video that explained that there's actually a shortcut command.
- Instead of using sudo packman -U, immediately after I used `sudo pacman -S base-devel`, I could use `makepkg -sic`

16. Installing zsh
- `sudo pacman -S zsh zsh-completions` This is installing zsh
- Enter zsh through `zsh`
- Setup key: ‘0’

17. Alias shortcuts
- `alias c='clear'` This will set a shorcut to simply type out `c` instead of `clear` to clear the terminal
- `alias ls='ls --color=auto'` This will set a shortcut to simply type `ls` and have it automatically be in color

18. Adding colors to bash
- Enter this command to copy the user-specific bashrc file, `cp .bashrc .bashrc.backup`
  - Or edit the one in the etc location: `sudo cp /etc/bash.bashrc /etc/bash.bashrc.backup`
- I chose to edit the main bashrc file by doing `sudo nano /etc/bash.bashrc`
- I then changed the PS1 variable to become: `PS1="${GREEN}KeepMovingForward${RESET}>"`
- But added in the following lines just above the PS1 variable
- `GREEN="\[$(tput setaf 2)\]"`
- `RESET="\[$(tput sgr0)\]"`
- Finally, after saving the changes, I typed out this command `source /etc/bash.bashrc`
