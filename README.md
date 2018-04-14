# Guide to install Simons Arch setup

This guide is mostly set up for myself so that I can learn more about the process. Any suggestions or improvements are welcome!

This guide assumes you have a live disk of Arch linux on a usb stick. Otherwise these can be found [here](https://www.archlinux.org/download/). It will teach you to install Arch Linux alongside an existing partition of Windows.

Don't forget to disable fast boot in Windows to ensure that files are saved in Windows when switching OS.

## Verifying internet and EFI is set correctly

Make sure that you have an ethernet cable and that EFI mode is set through BIOS.

### Send 3 packets to check internet
```shell
$ ping -c 3 google.com
```


### Check for efi variables. Should spit out a list
```shell
$ efivar -l
```


## Disk partitioning

I have a Windows install on the same disk. We will create a root partition, a boot partition and a swap partition.

### Checking the current disk partitioning

```shell
$ lsblk
```

This should give you a list of partitions. I have a ssd so for me is says:
sda
sda1
sda2
sda3
sda4
sda5
These are refered to as /dev/sdaN, where N is 1-5. The first four is Windows partitions, so the fifth will become the first associated with Linux.

### Create disk partitions for Linux.

```shell
$ cgdisk /dev/sda
```

You will be presented with a text based program to partition /dev/sda. Select the Partition Type *free space* and press New. Press Enter to choose the default start sector. Write '1024MiB' and choose code 'EF00' for EFI system. Name this partition 'boot'.
Make another partition for swap space. The size depends on your amount of RAM. For me, since I have 16 GiB I can safely choose 8GiB of swap space. The code for swap is '8200', and name is 'swap'.
Make one last partition with the rest of the sectors (press Enter two times to default to all sectors) and the default code '8300' and call it 'root'.
Take note of which partition is stored on which number.
You can now press Write and then Quit to exit the application.

### Creating the file systems

```shell
$ mkfs.fat -F32 /dev/sda5
$ mkswap /dev/sda6
$ swapon /dev/sda6
$ mkfs.ext4 /dev/sda7
```

This will create a bootable FAT32 on sda5. Swap on sda6, and ext4 file system on sda7.

## Mount and install Arch

### Mounting partitions

```shell
$ mount /dev/sda7 /mnt
$ mkdir /mnt/boot
$ mount /dev/sda5 /mnt/boot
```

### Setting up Arch repository mirrorlist

Find the closest mirror to optimize package installation. First make a backup of the current mirrorlist.

```shell
$ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Use *sed* to uncomment all of the mirrors (sed - stream editor).

```shell
$ sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup
```

Rank the top 6 mirrors and leave them uncommented. This will take a while.

```shell
$ rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

### Install Arch base and development files

```shell
$ pacstrap -i /mnt base base-devel vim bash-completion
```

### Generate fstab file

This will make sure that the mountpoints are remembered when rebooting later.

```shell
$ genfstab -U -p /mnt >> /mnt/etc/fstab
```

Make sure that every partition is there.

```shell
$ vim /mnt/etc/fstab
```

### Change root into your system and set up your locals

```shell
$ arch-chroot /mnt
```

For language create a locale.gen file.

```shell
$ vim /etc/locale.gen
```

Uncomment your locale. I uncommented en_US.UTF-8. Generate the locale.

```shell
$ locale-gen
```

Set your language

```shell
$ echo LANG=en_US.UTF-8 > /etc/locale.conf
$ export LANG=en_US.UTF-8
```

Set a symbolic link to your time zone. Each time zone can be found in /usr/share/zoneinfo/

```shell
$ ln -s /usr/share/zoneinfo/your-time-zone > /etc/localtime
```

Set hardware clock to utc

```shell
$ hwclock --systohc --utc
```

Synchronize the system clock across the network

```shell
$ timedatectl set-ntp true
```

Set up your hostname.

```shell
$ echo hostName > /etc/hostname
```

If you have a SSD then you might want to enable TRIM support.

```shell
$ systemctl enable fstrim.timer
```

### Setup pacman and user configuration

Open pacman.conf and uncomment support for multilib (if you want to install 32bit programs)

```shell
$ vim /etc/pacman.conf
```

Here you uncomment the two lines marked as
[multilib]
Include = /etc/pacman.d/mirrorlist

Save the file and update your system

```shell
$ pacman -Sy
```

Setup a root password

```shell
$ passwd
```

Now add users to the groups wheel,storage,power

```shell
$ useradd -m -g users -G wheel,storage,power -s /bin/bash someUsername
```

Set a password for that user

```shell
$ passwd someUsername
```

Setup which users can use the sudo command

```shell
$ EDITOR=vim visudo
```

Uncomment
%wheel ALL=(ALL) ALL

At the end of the file add
Defaults rootpw

Install the bootloader. First check that EFI variables are mounted.

```shell
$ mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

If it says that they are already mounted, you're good to go!

Install the bootloader

```shell
$ bootctl install
```

We need to know our root partition. Mine is sda7. You can see all partition by typing "lblk". Create an arch.conf file.

```shell
$ vim /boot/loader/entries/arch.conf
```

Add these lines:

```shell
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
```

Save and close this file. We need to add the root partition PARTUUID to this file. This is easiest done by a shell command.

```shell
$ echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/sda7) rw" >> /boot/loader/entries/arch.conf
```

Where you change /dev/sda7 to your root partition.

If you have an intel processor you need to install intel-ucode.

```shell
$ pacman -S intel-ucode
```

Then you need to add another line to the arch.conf

```shell
$ sudo vim /boot/loader/entries/arch.conf
```

Add the line "initrd /intel-ucode.img" before "initrd /initramfs-linux.img"

Now we need to enable dhcpcd at system start.

```shell
$ ip link
```

Check for your address. It looks like "enpXXsY".

```shell
$ sudo systemctl enable dhcpcd@enpXXsY.service
```

Download networkmanager and enable it.

```shell
$ sudo pacman -S networkmanager
$ sudo systemctl enable NetworkManager.service
```

Some programs need to have headers installed.

```shell
$ sudo pacman -S linux-headers
```

I like the open source drivers [Nouveau](https://wiki.archlinux.org/index.php/nouveau) as a driver for Nvidia cards.

```shell
$ sudo pacman -S xf86-video-nouveau lib32-mesa
```

#### Nvidia proprietary drivers

If you want the proprietary drivers you should install the following

```shell
$ sudo pacman -S nvidia libglvnd nvidia-utils opencl-nvidia nvidia-settings lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia
```

Now set the drm kernel modules for Nvidia

```shell
$ vim /etc/mkinitcpio.conf
```

Add the Nvidia modules to the MODULES entry as

```text
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

Now ensure that these are loaded on boot by adding nvidia-drm.modeset=1 to the root entry in /boot/loader/entries/arch.conf. This line should look something like this when you're done;

```text
options root=PARTUUID=... rw nvidia-drm.modeset=1
```

A big confusion about the Nvidia drivers is that we need to run mkinitcpio to recreate the initial ramdisk environment every time the kernel is upgraded. This is something I forget every time so we will add a pacman hook to do this automatically.

```shell
$ vim /etc/pacman.d/hooks/nvidia.hook
```

Add this hook;

```ini
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia

[Action]
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P
```

You should now be able to reboot your system. Exit to arch root, unmount all drives and reboot the system.

```shell
$ exit
$ umount -R /mnt
$ reboot
```

You should now be able to boot in and see a login screen.

If you are on a laptop you might want touchpad support

```shell
$ sudo pacman -S xf86-input-synaptics
```

Now install xorg as a background for i3 and mesa for 3d support.

```shell
$ sudo pacman -S xorg-server xorg-apps xorg-xinit xorg-twm xorg-xclock xterm mesa
```

Make sure that it works.

```shell
$ startx
```


It should run three terminals and a clock. Your mouse should now work aswell.

### Optimize your system

Install ccache to make compiling a lot faster. We will also set up the system to utilize all the cores. First, find out how many cores are available (this is the number of threads you have if your cpu supports hyperthreading).

```shell
$ lscpu
```


This command should show you a list of information about your cpu. Find the line called "CPU(s): XX". For me XX=12 since I have a AMD Ryzen 5 1600x. Install ccache.

```shell
$ sudo pacman -S ccache
```


Enable ccache and set our flags in makepkg.conf

```shell
$ sudo vim /etc/makepkg.conf
```


Find the line with "BUILDENV='... !ccache ...'. Remove ! in front of ccache to enable it. Find the line with "MAKEFLAGS=" and change it into
MAKEFLAGS="-j13 -l12"
Where -j(XX+1) -lXX, where XX is the number you got from the "lscpu" command earlier.

Utilize the optimization outside of the package managers.

```shell
$ vim ~/.bashrc
```


Add these lines to the end of the file
export PATH="/usr/lib/ccache/bin/:$PATH"
export MAKEFLAGS="-j13 -l12"
Where you of course change -j and -l to your values.

Now we can install our terminal emulator. I prefer [urxvt](https://wiki.archlinux.org/index.php/Rxvt-unicode). urxvt-perls are some preferred librarier (url-select, keyboard-select).

```shell
$ pacman -S rxvt-unicode urxvt-perls
```


Download zsh

```shell
$ pacman -S zsh zsh-completions
```


Finally ssh for remote access

```shell
$ pacman -S openssh
```


## Install yay for AUR repositories

### Install git to be able to download yay, and jshon and wget to use it
```shell
$ pacman -S git jshon wget
```


### Download and install yay

```shell
$ git clone https://aur.archlinux.org/yay.git
$ cd ./yay
$ makepkg -s
$ sudo pacman -U *xz
```

### Delete the yay directory as it is installed

```shell
$ cd ..
$ rm -rf yay
```

Now that yay is set up we can download scripts to synchronize our local time with servers keeping time for our timezone (set in /usr/share/zoneinfo/your-time-zone previously). ntpd works without configuration so you only have to install it.

```shell
$ sudo pacman -S ntp
```

Synch with the servers.

```shell
$ sudo ntpd -qg
$ sudo hwclock --systohc
```

A nice way of keeping track of this is by enabling it to run every time a network interface is available. For NetworkManager, the hooks are called dispatcher scripts, so lets enable them to launch at boot (--now starts the service now aswell).

```shell
$ systemctl enable --now NetworkManager-dispatcher.service
```

Now the ntpd script can be downloaded (it is automatically configured. Note that the dispatcher scripts must be owned by root).

```shell
$ yay -S networkmanager-dispatcher-ntpd
```

## Install i3-gaps as a window manager

### First install i3 to get the dependencies

```shell
$ pacman -S i3-gaps
```

### Perl scripts to interact with i3
```shell
$ pacman -S perl-anyevent-i3 perl-json-xs
```


### Set i3 to start on startx command
```shell
$ echo "exec i3" > ~/.xinitrc
```


## Make the system more user friendly

### X11 scripting tool
```shell
$ pacman -S xdotool
```

### Screen compositor
```shell
$ pacman -S compton
```

### Resize font on the fly
```shell
$ yay -S urxvt-resize-font-git
```


### Font
```shell
$ yay -S ttf-iosevka ttf-croscore
```


### Fade out cursor
```shell
$ yay -S unclutter-xfixes-git
```


### Project indexer
```shell
$ pacman -S ctags
```


### Terminal multiplexor
```shell
$ pacman -S tmux
```


### Sound control (jack for sink detection and pavucontrol for GUI overview)
```shell
$ pacman -S pulseaudio pulseaudio-jack pavucontrol
```


### Configure multiple monitor with arandr. It creates xrandr configuration files from a GUI.
Save the config file and add it to your i3 config file as an "exec". This way the configuration will be used each time you log in to your system.
```shell
$ pacman -S arandr
```


### Application launcher
Start this application with "rofi -show run"
```shell
$ yay -S rofi
```


### Watching files for changes and running commands on events
```shell
$ yay -S entr
```


### For setting background
```shell
$ pacman -S feh
```


### Adjusts the color temperature of your screen
```shell
$ pacman -S redshift
```


### PDF-reader
```shell
$ pacman -S mupdf
```


### Browser, (gst for playing youtube videos)
```shell
$ pacman -S qutebrowser
$ pacman -S gst-plugins-{base,good,bad,ugly} gst-libav
```


### Password manager
```shell
$ pacman -S pass gnupg
```


### File manager
```shell
$ pacman -S ranger
```


### Statur bar
```shell
$ yay -S polybar
```

### Polybar need wireless-tools to use eth/wlan modules
```shell
$ pacman -S wireless_tools
```

### For tk windows in plotting programs such as matplotlib from python
```shell
$ pacman -S tk
```


### For youcompleteme's autocompletion windows
```shell
$ yay -S ncurses5-compat-libs
```


### For latex
```shell
$ pacman -S texlive-most texlive-lang
```


### For latex compilation
```shell
$ yay -S rubber
```


### Taskwarrior for task management
```shell
$ pacman -S task
```


### Maim for screenshots
```shell
$ pacman -S maim
```


### Changing document format from one to another
```shell
$ pacman -S pandoc
```


### Node.js and npm for javascript development
```shell
$ pacman -S npm
```


### tern for javascript autocompletion
```shell
$ npm install -g tern
```

### Install package manager for python
```shell
$ pacman -S python-pip
```

### For downloading youtube videos and also some fancy mpv i3 scripting
```shell
$ pip install youtube-dl
```


### For formatting of html, css and javascript
```shell
$ pip install jsbeautifier
```


### tmux project manager
```shell
$ pip install tmuxp
```


### For lintint code
```shell
$ pacman -S shellcheck
$ pip install vim-vint
```

### Drawing and designing
```shell
$ pacman -S blender inkscape gimp
```

