# Setup Raspberry Pi OS

First Install Raspberry Pi OS to the SD Card using [Raspberry Pi Imager](https://www.raspberrypi.org/software/). Insert that SD card into the PI and walk through the base setup of user/password, Country, Keyboard, and Timezone. 

Next, go to the menu and select Pi Configuration and set the following items: 
- System -> Boot  -> To CLI
- Interfaces tab -> SSH -> enable

Reboot and attempt to SSH in. If you can't find the pi on the network, try a tool like [Angry IP Scanner](https://angryip.org/).

> SSH into the Pi and run `passwd` to enter in a new password for the pi. 

---

## Install Packages and Setup File Structure

Now that you can SSH into the PI. Run the following command to install additional packages:

```shell 
sudo apt-get update && sudo apt-get install dnsmasq pxelinux syslinux-efi wget
```

Create the following folders: 

```shell
 sudo mkdir -p /mnt/data/netboot/{bios,efi64}
 sudo mkdir /mnt/data/isos
 sudo mkdir -p /mnt/data/netboot/boot/amd64/fedora/33
 sudo mkdir /mnt/data/netboot/ks.cfg
```

Copy the appropriate syslinux libraries to their directories. 
```shell
$ cp \
  /usr/lib/syslinux/modules/bios/{ldlinux,vesamenu,libcom32,libutil}.c32 \
  /usr/lib/PXELINUX/pxelinux.0 \
  /mnt/data/netboot/bios

$ cp \
  /usr/lib/syslinux/modules/efi64/ldlinux.e64 \
  /usr/lib/syslinux/modules/efi64/{vesamenu,libcom32,libutil}.c32 \
  /usr/lib/SYSLINUX.EFI/efi64/syslinux.efi \
  /mnt/data/netboot/efi64
```

Download the Fedora ISO to `mnt/data/isos`
```shell
cd /mnt/data/isos
wget https://download.fedoraproject.org/pub/fedora/linux/releases/33/Server/x86_64/iso/Fedora-Server-dvd-x86_64-33-1.2.iso
```

Mount the ISO and copy the files to the destination directory
```shell
sudo mount -o loop -t iso9660 /mnt/data/isos/Fedora-Server-dvd-x86_64-33-1.2.iso /media

sudo rsync -av /media/ /mnt/data/netboot/boot/amd64/fedora/33

sudo unmount /media
```


Use a [Fedora Kickstart File](https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/advanced/Kickstart_Installations/) to Automate the install process by copying the ks.cfg file into the `/mnt/data/netboot/ks.cfg/` directory. Be sure to user/password in that file (see Create User Account Documentation)[https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-user].
