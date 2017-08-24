# Guide to install Simons Arch setup

This guide is mostly set up for myself so that I can learn more about the process. Any suggestions or improvements are welcome!

This guide assumes you have a live disk of Arch linux on a usb stick. Otherwise these can be found [here](https://www.archlinux.org/download/). It will teach you to install Arch Linux alongside an existing partition of Windows. It will utilize Grub to switch between these. Large thanks to [gloriouseggroll](https://www.gloriouseggroll.tv/) for large parts of this guide.

* ## Verifying internet and EFI is set correctly

---

Make sure that you have an ethernet cable and that EFI mode is set through BIOS.

   * ### Send 3 packets to check internet
         $ ping -c 3 google.com

   * ### Check for efi variables. Should spit out a list
         $ efivar -l

* ## Disk partitioning

---

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
        Where the first four is Windows, so the fifth will become Linux. These are refered to as /dev/sdaN, where N is 1-5.

   * ### Create disk partitions for Linux.
         $ cgdisk /dev/sda

* ## Install apacman for AUR repositories

---

   * ### Install git to be able to download apacman
         $ pacman -S git

   * ### Download apacman, install apacman
         $ git clone https://github.com/oshazard/apacman.git
         $ cd ./apacman
         $ ./apacman -S apacman

   * ### Delete the apacman directory as it is installed
         $ cd
         $ sudo -R apacman

* ## Install i3-gaps as a window manager

---

   * ### First install i3 to get the dependencies
         $ pacman -S i3

   * ### Use apacman to download i3-gaps and remove i3wm when asked
         $ apacman -S i3-gaps

   * ### Set i3 to start on startx command
         $ echo "exec i3" > ~/.xinitrc
