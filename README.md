# Guide to install Simons Arch setup

This guide is mostly set up for myself so that I can learn more about the process. Any suggestions or improvements are welcome!

This guide assumes you have a live disk of Arch linux on a usb stick. Otherwise these can be found [here](https://www.archlinux.org/download/). It will teach you to install Arch Linux alongside an existing partition of Windows.

Don't forget to disable fast boot in Windows to ensure that files are saved in Windows when switching OS.

* ## Verifying internet and EFI is set correctly

Make sure that you have an ethernet cable and that EFI mode is set through BIOS.

   * ### Send 3 packets to check internet
         $ ping -c 3 google.com

   * ### Check for efi variables. Should spit out a list
         $ efivar -l

* ## Disk partitioning

I have a Windows install on the same disk. We will create a root partition, a boot partition and a swap partition.

   * ### Checking the current disk partitioning

         $ lsblk

         This should give you a list of partitions. I have a ssd so for me is says:
         sda
            sda1
            sda2
            sda3
            sda4
            sda5
        These are refered to as /dev/sdaN, where N is 1-5. The first four is Windows partitions, so the fifth will become the first associated with Linux.

   * ### Create disk partitions for Linux.

         $ cgdisk /dev/sda

         You will be presented with a text based program to partition /dev/sda. Select the Partition Type *free space* and press New. Press Enter to choose the default start sector. Write '1024MiB' and choose code 'EF00' for EFI system. Name this partition 'boot'.
         Make another partition for swap space. The size depends on your amount of RAM. For me, since I have 16 GiB I can safely choose 8GiB of swap space. The code for swap is '8200', and name is 'swap'.
         Make one last partition with the rest of the sectors (press Enter two times to default to all sectors) and the default code '8300' and call it 'root'.
         Take note of which partition is stored on which number.
         You can now press Write and then Quit to exit the application.

   * ### Creating the file systems

         $ mkfs.fat -F32 /dev/sda5
         $ mkswap /dev/sda6
         $ swapon /dev/sda6
         $ mkfs.ext4 /dev/sda7

         This will create a bootable FAT32 on sda5. Swap on sda6, and ext4 file system on sda7.

* ## Mount and install Arch

   * ### Mounting partitions

         $ mount /dev/sda7 /mnt
         $ mkdir /mnt/boot
         $ mount /dev/sda5 /mnt/boot

   * ### Setting up Arch repository mirrorlist

         Find the closest mirror to optimize package installation. First make a backup of the current mirrorlist.

         $ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

         Use *sed* to uncomment all of the mirrors (sed - stream editor).

         $ sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup

         Rank the top 6 mirrors and leave them uncommented. This will take a while.

         $ rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist

   * ### Install Arch base and development files

         $ pacstrap -i /mnt base base-devel

   * ### Generate fstab file

         This will make sure that the mountpoints are remembered when rebooting later.

         $ genfstab -U -p /mnt >> /mnt/etc/fstab

         Make sure that every partition is there.

         $ nano /mnt/etc/fstab

   * ### Change root into your system and set up your locals

         $ arch-chroot /mnt

         For language create a locale.gen file.

         $ nano /etc/locale.gen

         Uncomment your locale. I uncommented en_US.UTF-8. Generate the locale.

         $ locale-gen

         Set your language

         $ echo LANG=en_US.UTF-8 > /etc/locale.conf
         $ export LANG=en_US.UTF-8

         Set a symbolic link to your time zone. Each time zone can be found in /usr/share/zoneinfo/

         $ ln -s /usr/share/zoneinfo/your-time-zone > /etc/localtime

         Set hardware clock to utc

         $ hwclock --systohc --utc

         Set up your hostname.

         $ echo hostName > /etc/hostname

         If you have a SSD then you might want to enable TRIM support.

         $ systemctl enable fstrim.timer

   * ### Setup pacman and user configuration

         Open pacman.conf and uncomment support for multilib (if you want to install 32bit programs)

         $ nano /etc/pacman.conf

         Here you uncomment the two lines marked as
         [multilib]
         Include = /etc/pacman.d/mirrorlist

         Save the file and update your system

         $ pacman -Sy

         Setup a root password

         $ passwd

         Now add users to the groups wheel,storage,power

         $ useradd -m -g users -G wheel,storage,power -s /bin/bash someUsername

         Set a password for that user

         $ passwd someUsername

         Setup which users can use the sudo command

         $ EDITOR=nano visudo

         Uncomment
         %wheel ALL=(ALL) ALL

         At the end of the file add
         Defaults rootpw

         Install the bootloader. First check that EFI variables are mounted.

         $ mount -t efivarfs efivarfs /sys/firmware/efi/efivars

         If it says that they are already mounted, you're good to go!

         Install the bootloader

         $ bootctl install

         We need to know our root partition. Mine is sda7. You can see all partition by typing "lblk". Create an arch.conf file.

         $ nano /boot/loader/entries/arch.conf

         Add these lines:

         title Arch Linux
         linux /vmlinuz-linux
         initrd /initramfs-linux.img

         Save and close this file. We need to add the root partition PARTUUID to this file. This is easiest done by a shell command.

         $ echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/sda7) rw" >> /boot/loader/entries/arch.conf

         Where you change /dev/sda7 to your root partition.

             If you have an intel processor you need to install intel-ucode.

             $ pacman -S intel-ucode

             Then you need to add another line to the arch.conf

             $ sudo nano /boot/loader/entries/arch.conf

             Add the line "initrd /intel-ucode.img" before "initrd /initramfs-linux.img"

         Now we need to enable dhcpcd at system start.

         $ ip link

         Check for your address. It looks like "enpXXsY".

         $ sudo systemctl enable dhcpcd@enpXXsY.service

         Download networkmanager and enable it.

         $ sudo pacman -S networkmanager
         $ sudo systemctl enable NetworkManager.service

         Some programs need to have headers installed.

         $ sudo pacman -S linux-headers

         I like the open source drivers [Nouveau](https://wiki.archlinux.org/index.php/nouveau) as a driver for Nvidia cards.

         $ sudo pacman -S xf86-video-nouveau lib32-mesa

         You should now be able to reboot your system. Exit to arch root, unmount all drives and reboot the system.

         $ exit
         $ umount -R /mnt
         $ reboot

         You should now be able to boot in and see a login screen.

         Now install xorg as a background for i3.

         $ sudo pacman -S xorg-server xorg-apps xorg-xinit xorg-twm xorg-xclock xterm

         Make sure that it works.

         $ startx

         It should run three terminals and a clock. Your mouse should now work aswell.

   * ### Optimize your system

         Install ccache to make compiling a lot faster. We will also set up the system to utilize all the cores. First, find out how many cores are available (this is the number of threads you have if your cpu supports hyperthreading).

         $ lscpu

         This command should show you a list of information about your cpu. Find the line called "CPU(s): XX". For me XX=12 since I have a AMD Ryzen 5 1600x. Install ccache.

         $ sudo pacman -S ccache

         Enable ccache and set our flags in makepkg.conf

         $ sudo nano /etc/makepkg.conf

         Find the line with "BUILDENV='... !ccache ...'. Remove ! in front of ccache to enable it. Find the line with "MAKEFLAGS=" and change it into
         MAKEFLAGS="-j13 -l12"
         Where -j(XX+1) -lXX, where XX is the number you got from the "lscpu" command earlier.

         Utilize the optimization outside of the package managers.

         $ nano ~/.bashrc

         Add these lines to the end of the file
         export PATH="/usr/lib/ccache/bin/:$PATH"
         export MAKEFLAGS="-j13 -l12"
         Where you of course change -j and -l to your values.

         Now we can install our terminal emulator. I prefer [urxvt](https://wiki.archlinux.org/index.php/Rxvt-unicode). urxvt-perls are some preferred librarier (url-select, keyboard-select).

         $ pacman -S rxvt-unicode urxvt-perls
         $ apacman -S urxvt-resize-font-git

	 Download zsh

	 $ pacman -S zsh zsh-completions

	 Finally ssh for remote access
	 
	 $ pacman -S openssh

* ## Install apacman for AUR repositories

   * ### Install git to be able to download apacman, and jshon and wget to use it
         $ pacman -S git jshon wget

   * ### Download apacman, install apacman
         $ git clone https://github.com/oshazard/apacman.git
         $ cd ./apacman
         $ ./apacman -S apacman

   * ### Delete the apacman directory as it is installed
         $ cd ..
         $ sudo rm -R apacman

* ## Install i3-gaps as a window manager

   * ### First install i3 to get the dependencies
         $ pacman -S i3

   * ### Use apacman to download i3-gaps and remove i3wm when asked
         $ apacman -S i3-gaps

   * ### Perl scripts to interact with i3
         $ pacman -S perl-anyevent-i3 perl-json-xs

   * ### Set i3 to start on startx command
         $ echo "exec i3" > ~/.xinitrc

* ## Make the system more user friendly

   * ### Font
         $ apacman -S ttf-iosevka ttf-croscore

   * ### Fade out cursor
         $ pacman -S unclutter

   * ### Terminal multiplexor
         $ pacman -S tmux

   * ### Sound control
         $ pacman -S pulseaudio

   * ### Configure multiple monitor with arandr. It creates xrandr configuration files from a GUI.
         Save the config file and add it to your i3 config file as an "exec". This way the configuration will be used each time you log in to your system.
         $ pacman -S arandr

   * ### Application launcher
         Start this application with "rofi -show run"
         $ apacman -S rofi

   * ### For setting background
         $ pacman -S feh

   * ### Adjusts the color temperature of your screen
         $ pacman -S redshift

   * ### PDF-reader
         $ pacman -S mupdf

   * ### Browser, (gst for playing youtube videos)
         $ pacman -S qutebrowser
         $ pacman -S gst-plugins-{base,good,bad,ugly} gst-libav

   * ### Password manager
         $ pacman -S pass gnupg

   * ### File manager
         $ pacman -S ranger

   * ### Statur bar
         $ apacman -S polybar

   * ### Editor
         $ pacman -S vim

   * ### For tk windows in plotting programs such as matplotlib from python
         $ pacman -S tk

   * ### For youcompleteme's autocompletion windows
         $ apacman -S ncurses5-compat-libs

   * ### For latex
         $ pacman -S texlive-most texlive-lang

   * ### For latex compilation
         $ apacman -S rubber

   * ### Offline documentation browser
         $ pacman -S zeal

   * ### Taskwarrior for task management
         $ pacman -S task

   * ### Maim for screenshots
         $ pacman -S maim
