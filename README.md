# Guide to install Simons Arch setup

This guide is mostly set up for myself so that I can learn more about the process. Any suggestions or improvements are welcome!

This guide assumes you have a live disk of Arch linux on a usb stick. Otherwise these can be found [here](https://www.archlinux.org/download/). It will teach you to install Arch Linux alongside an existing partition of Windows. It will utilize Grub to switch between these. Large thanks to [gloriouseggroll](https://www.gloriouseggroll.tv/) for large parts of this guide.

* ## Verifying internet and EFI is set correctly

Make sure that you have an ethernet cable and that EFI mode is set through BIOS.

   * ### Send 3 packets to check internet
         $ ping -c 3 google.com

   * ### Check for efi variables. Should spit out a list
         $ efivar -l

* ## Disk partitioning

I have a Windows install on the same disk. We will create a root partition, a boot partition and a swap partition. No grub partition will be created since according to [Arch wiki](https://wiki.archlinux.org/index.php/GRUB) you can use the efi partition created by Windows.

   * ### Checking the current disk partitioning

         $ lblk

         This should give you a list of partitions. I have a ssd so for me is says:
         sda
            sda1
            sda2
            sda3
            sda4
            sda5
        Where the first four is Windows partitions, so the fifth will become Linux. These are refered to as /dev/sdaN, where N is 1-5.

   * ### Create disk partitions for Linux.

         $ cgdisk /dev/sda

         You will be presented with a text based program to partition /dev/sda. Select the Partition Type *free space* and press New. Press Enter to choose the default start sector. Write '1024MiB' and choose code 'EF00' for EFI system. Name this partition 'boot'.
         Make another partition for swap space. The size depends on your amount of RAM. For me, since I have 16 GiB I can safely choose 8GiB of swap space. The code for swap is '8200', and name is 'swap'.
         Make one last partition with the rest of the sectors (press Enter two times to default to all sectors) and the default code '8300' and call it 'root'.
         Take note of which partition is stored on which number. Also note down the number of the Windows partition that is described with the Partition Type *EFI System*. It is usually arount 100 MiB. Mine is /dev/sda2. This will be used by Grub later.
         You can now press Write and then Quit to exit the application.

   * ### Creating the file systems

         $ mkfs.fat -F32 /dev/sda5
         $ mkswap /dev/sda6
         $ swapon /dev/sda6
         $ mkfs.ext4 /dev/sda7

         This will create a bootable FAT32 on sda5. Swap on sda6, and ext4 file system on sda7.

* ## Mount and install Arch

   * ### Mounting partitions
         
         Remember the number of the EFI system on windows. We will create a connection between Windows and Linux there that Grub will use. For me, this partition is /dev/sda2.

         $ mount /dev/sda7 /mnt
         $ mkdir /mnt/boot
         $ mount /dev/sda5 /mnt/boot
         $ mkdir /mnt/boot/efi
         $ mount /dev/sda2 /mnt/boot/efi

   * ### Setting up Arch repository mirrorlist
         
         Find the closest mirror to optimize package installation. First make a backup of the current mirrorlist.

         $ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

         Use *sed* to uncomment all of the mirrors (sed - stream editor).

         $ sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup

         Rank the top 6 mirrors and leave them uncommented. This will take a while.
         
         $ rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist

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

   * ### Set i3 to start on startx command
         $ echo "exec i3" > ~/.xinitrc
