# Guide to install Simons Arch setup

* ## Install apacman for AUR repositories --

   * ### Install git to be able to download apacman
pacman -S git

   * ### Download apacman, install apacman
git clone https://github.com/oshazard/apacman.git
cd ./apacman
./apacman -S apacman

   * ### Delete the apacman directory as it is installed
cd ..
sudo -R apacman

* ## Install i3-gaps as a window manager

   * ### First install i3 to get the dependencies
pacman -S i3

   * ### Use apacman to download i3-gaps and remove i3wm when asked
apacman -S i3-gaps

   * ### Set i3 to start on startx command
echo "exec i3" > ~/.xinitrc
