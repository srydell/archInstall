# Guide to install Simons Arch setup

This guide is mostly set up for myself so that I can learn more about the process. Any suggestions or improvements are welcome!

This guide assumes you have a live disk of Arch linux on a usb stick. Otherwise these can be found [here](https://www.archlinux.org/download/). It will teach you to install Arch Linux alongside an existing partition of Windows.

Don't forget to disable fast boot in Windows to ensure that files are saved in Windows when switching OS.

Instructions that are specific to a set of hardware/not something every user will want to do are surrounded by bars. An example:

---

Install program only used by coffee drinkers:

```shell
$ pacman -S pour-me-coffee
```

---

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

This should give you a list of partitions. It should say something like:

sda
sda1
sda2
sda3
sda4
sda5

The first four in my case are the Windows partitions, so the fifth will become the first associated with Linux.

### Create disk partitions for Linux.

```shell
$ cgdisk /dev/sda
```

You will be presented with a text based program to partition /dev/sda. Select the Partition Type *free space* and press New. Press Enter to choose the default start sector. Write `1024MiB` and choose code `EF00` for EFI system. Name this partition `boot`. Repeat this until you have the folowing:

* Size: 1024MiB, Code: EF00, Name: boot
* Size: 8GiB, Code: 8200, Name: swap
* Size: 25GiB, Code: 8300, Name: root
* Size: -, Code: 8300, Name: home

To make the home partition, just press enter when asked for size to get the remaining space on the disk. I recommend using a separate partition for root where all applications are stored. This way if you ever corrupt your system, you can just reinstall the root partition.
Take note of which partition is stored on which number.
You can now press Write and then Quit to exit the application.

### Creating the file systems

```shell
$ mkfs.fat -F32 /dev/sda5
$ mkswap /dev/sda6
$ swapon /dev/sda6
$ mkfs.ext4 /dev/sda7
$ mkfs.ext4 /dev/sda8
```

This will create a bootable FAT32 on sda5 (boot). Swap on sda6 (swap), and ext4 file system on sda7 and sda8 (root and home).

## Mount and install Arch

### Mounting partitions

```shell
$ mount /dev/sda7 /mnt
$ mkdir /mnt/{boot,home}
$ mount /dev/sda5 /mnt/boot
$ mount /dev/sda8 /mnt/home
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

Uncomment your locale. For example: `en_US.UTF-8`. Generate the locale.

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

Set up your hostname.

```shell
$ echo hostName > /etc/hostname
```

---

If you have a SSD then you might want to enable TRIM support.

```shell
$ systemctl enable fstrim.timer
```

---

### Setup pacman and user configuration

Open `pacman.conf` and uncomment support for multilib (if you want to install 32bit programs)

```shell
$ vim /etc/pacman.conf
```

Here you uncomment the two lines marked as

```ini
...
[multilib]
Include = /etc/pacman.d/mirrorlist
...
```

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

```ini
...
%wheel ALL=(ALL) ALL
...
```

At the end of the file add
```ini
...
Defaults rootpw
...
```

Install the bootloader. First check that EFI variables are mounted.

```shell
$ mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

If it says that they are already mounted, you're good to go!

Install the bootloader

```shell
$ bootctl install
```

Create an arch.conf file.

```shell
$ vim /boot/loader/entries/arch.conf
```

Add these lines:

```shell
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
```

Save and close the file. We need to know our root partition. Mine is sda7. You can see all partition by typing `lsblk`. Add the root partition PARTUUID to the arch.conf file. This is easiest done by a shell command.

```shell
$ echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/sda7) rw" >> /boot/loader/entries/arch.conf
```

---

If you have an intel processor you need to install intel-ucode.

```shell
$ pacman -S intel-ucode
```

Then you need to add another line to the arch.conf

```shell
$ sudo vim /boot/loader/entries/arch.conf
```

Add the intel-ucode image:

```ini
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
```

---

Now we need to enable dhcpcd at system start.

```shell
$ ip link
```

Check for your address. It looks like `enpXXsY`.

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

---

#### Nvidia drivers

##### Open source drivers - Nouveau

I like the open source drivers [Nouveau](https://wiki.archlinux.org/index.php/nouveau) since they are much easier to install. If you want to play games or use blender and use such gpu intensive programs, you might want to install the proprietary drivers. To install Nouveau with 32 bit support type

```shell
$ sudo pacman -S xf86-video-nouveau lib32-mesa
```

##### Proprietary Nvidia drivers

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

Now ensure that these are loaded on boot by adding `nvidia-drm.modeset=1` to the root entry in `/boot/loader/entries/arch.conf`. This line should look something like this when you're done;

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

---

You should now be able to reboot your system. Exit to arch root, unmount all drives and reboot the system.

```shell
$ exit
$ umount -R /mnt
$ reboot
```

You should now be able to boot in and see a login screen.

---
If you are on a laptop you might want touchpad support

```shell
$ sudo pacman -S xf86-input-synaptics
```
---

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

This command should show you a list of information about your cpu. Find the line called `CPU(s): XX`. For example `XX=12`. Install ccache.

```shell
$ sudo pacman -S ccache
```

Enable ccache and set our flags in makepkg.conf

```shell
$ sudo vim /etc/makepkg.conf
```

Find the line with `BUILDENV='... !ccache ...'`. Remove ! in front of ccache to enable it. Add the number of cores to makeflags. In the end you should have something like:

```ini
...
BUILDENV='... ccache ...'
...
MAKEFLAGS="-j13 -l12"
...
```

Where `-j(XX+1) -lXX`, where XX is the number you got from the `lscpu` command earlier.

Utilize the optimization outside of the package managers.

```shell
$ vim ~/.bashrc
```

Add these lines to the end of the file

```shell
export PATH="/usr/lib/ccache/bin/:$PATH"
export MAKEFLAGS="-j13 -l12"
```

Where you of course change -j and -l to your specific values.

Now we can install our terminal emulator. I prefer [urxvt](https://wiki.archlinux.org/index.php/Rxvt-unicode). urxvt-perls is a collection of utility libraries (url-select, keyboard-select).

```shell
$ pacman -S rxvt-unicode urxvt-perls
```

Also install ssh for remote access

```shell
$ pacman -S openssh keychain
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

A nice way of keeping track of this is by enabling it to run every time a network interface is available. For NetworkManager, the hooks are called dispatcher scripts, so lets enable them to launch at boot (`--now` starts the service aswell).

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

### Fonts

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
Save the config file and add it to your i3 config file as an `exec`. This way the configuration will be used each time you log in to your system.

```shell
$ pacman -S arandr
```

### Application launcher

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
$ pacman -S gst-plugins-{base,good,bad,ugly} gst-libav
$ pacman -S qutebrowser firefox-developer-edition
```

### Password manager

```shell
$ pacman -S pass gnupg
```

### File manager

```shell
$ pacman -S ranger
```

### Statusbar - Polybar need wireless-tools before compiling to use eth/wlan modules

```shell
$ pacman -S wireless_tools
$ yay -S polybar
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

### Viewing markdown files in the browser

```shell
$ pip install grip
```

### For linting code

```shell
$ pacman -S shellcheck
$ pacman -S python-pylint
$ pip install vim-vint
$ pip install proselint
```

### Drawing and designing

```shell
$ pacman -S blender inkscape gimp
```

### Ripgrep as a backend for fuzzy finder fzf (installed via minpac)

```shell
$ pacman -S ripgrep
```
