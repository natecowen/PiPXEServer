# Pi PXE Server

Overview: This repo shows how to setup a Pi PXE Server. PXE is useful for network installations. In this case it's being used to batch install Fedora 33 to my home lab cluster of 5 Intel Nuc (NUC5PPYB). 

---

## Setup Raspberry Pi OS

First Install Raspberry Pi OS to the SD Card using [Raspberry Pi Imager](https://www.raspberrypi.org/software/). Insert that SD card into the PI and walk through the base setup of user/password, Country, Keyboard, and Timezone. 

Next, go to the menu and select Pi Configuration and set the following items: 
- System -> Boot  -> To CLI
- Interfaces tab -> SSH -> enable

Reboot and attempt to SSH in. If you can't find the pi on the network, try a tool like [Angry IP Scanner](https://angryip.org/).

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


Use a [Fedora Kickstart File](https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/advanced/Kickstart_Installations/) to Automate the install process by copying the basic.cfg file into the /mnt/data/netboot/boot/amd64/fedora/33 directory. Be sure to user/password in that file (see Create User Account Documentation)[https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/appendixes/Kickstart_Syntax_Reference/#sect-kickstart-commands-user].


---

## Create A Boot Menu

Next, create a boot menu inside the tftp root. 

```shell
MENU TITLE PXE Boot Menu
DEFAULT vesamenu.c32

        LABEL local
                MENU LABEL  Boot from ^local drive
                LOCALBOOT 0xffff

        MENU BEGIN amd64
        MENU TITLE amd64

                MENU BEGIN Fedora
                MENU TITLE Fedora

                        LABEL server
                                MENU LABEL ^Install Fedora Server 33 64-bit (Web Install)
                                kernel ::boot/amd64/fedora/33/images/pxeboot/vmlinuz
                                append initrd=::boot/amd64/fedora/33/images/pxeboot/initrd.img inst.stage2=https://download.fedoraproject.org/pub/fedora/linux/releases/33/Server/x86_64/os ip=dhcp

                        LABEL server kickstarted
                                MENU LABEL ^Install Fedora 33 ( Kickstarted )
                                kernel ::boot/amd64/fedora/33/isolinux/vmlinuz
                                append initrd=::boot/amd64/fedora/33/isolinux/initrd.img inst.cmdline ip=dhcp inst.repo=nfs:192.168.X.X:/export/boot/amd64/fedora/33 inst.stage2=nfs:192.168.X.X:/export/boot/amd64/fedora/33 inst.ks=nsf:192.168.X.X:/export/boot/amd64/fedora/33:/ks/



                MENU END
        MENU END

```

> The :: symbol above allows us to reference the tftp root in an absolute way. 

Ensure the current working directory is `/mnt/data/netboot` and run `ln -rs pxelinux.cfg bios && ln -rs pxelinux.cfg efi64`. This will link the menu configs for both bios and efi64 together. 

> From LinuxConfig.org article in Helpful Links Section: Here we used the -r option of the ln command to create relative symbolic links. 

---

## Configure DNSMasq

`cd /etc` to configure the DNSMasq. Use `sudo nano dnsmasq.conf` to edit the configuration
```shell
# Turn of DNS Functionality
port=0
# Specify the network for DHCP
interface=eth0
#Set the DNS range via the subnet address, followed by "proxy". 
#This allows router to serve DHCP and avoid conflicts
dhcp-range=192.168.X.X,proxy

#enable tftp and set tftp-root
enable-tftp
tftp-root=/mnt/data/netboot

#Specify the client architect and how to traffic cop their request
pxe-service=x86PC,"PXELINUX (BIOS)",bios/pxelinux
pxe-service=x86-64_EFI,"PXELINUX (EFI)",efi64/syslinux.efi

#optionally add the logging queries
log-queries
log-facility=/var/log/dnsmasq.log

# save and exit the file
# restart the service via: 
sudo systemctl restart dnsmasq
```

--- 

## Add & Configure the Firewall


```shell
# install 
sudo apt-get update
sudo apt install ufw

# Add firewall rules
sudo ufw limit ssh
sudo ufw allow 67/udp
sudo ufw allow 69/udp
sudo ufw allow 4011/udp

sudo ufw allow from 192.168.x.0/24 to any port nfs
sudo ufw allow from 192.168.x.0/24 to any port 111
sudo ufw allow from 192.168.x/0/24 to any port nfs

# List current firewall rules
sudo ufw show added

# enable firewall
sudo ufw enable
```

---

## Install & Setup NFS Server

```shell
# install
sudo apt install nfs-kernel-server

# Make Export Directory
sudo mkdir -p /export/boot

# Set permissions
sudo chmod -R 0777 /export/boot

# Bind Mount the Directory
sudo nano /etc/fstab

# add the following to fstab to ensure mounts work after reboot and save
/mnt/data/netboot/boot    /export/boot   none    bind  0  0

# Mount and verify fstab mount works
sudo mount -a

# Verify mount via df
df -h /export/boot


# add our directories to local network: 
sudo nano /etc/exports

# add and save the following lines (you can do a whole subnet via something like 192.168.100.0)
/export       192.168.X.X/24(rw,nohide,insecure,no_subtree_check,async)
/export/boot 192.168.X.X/24(rw,nohide,insecure,no_subtree_check,async)
/export/boot/amd64/fedora/33 192.168.X.X/24(rw,nohide,insecure,no_subtree_check,async)
/export/boot/amd64/fedora/33/kickstart 192.168.X.0/24(rw,nohide,insecure,no_subtree_check,async)


#  Export the shared directory
sudo exportfs -a

# restart the service
sudo systemctl restart nfs-kernel-server

# Add a rule for firewall and nfs
sudo ufw allow from [clientIP or clientSubnetIP] to any port nfs

```

> If you are running into issues where the NFS connection for inst.repo are blocked. Verify the logs at /var/log/ufw.log with `tail -f -n 25 ufw.log`. Look for DPT (destination port), SPT (source port), DST (destination ip) & SRC (source ip). This will help to troubleshoot if the firewall is getting in the way.

I was still having port issues with NFS and had to use the edit the nfs-kernel-server file at `/etc/default`. NFS was assigning a random high level port that was being blocked. 

```shell
sudo nano /etc/default/nfs-kernel-server
# comment out the following line
#RPCMOUNTDOPTS="--manage-gids"

#force RPC to use a specific port. Any will do. Make it high
RPCMOUNTDOPTS="--port 10203" 

# Add a rull to the firewall for that port
sudo ufw allow from 192.168.100/24 to any port 10203

sudo service nfs-kernel-server restart

```

> For more info see this: [NFS Troubleshooting Raspberry Pi](https://blog.kevinckurtz.com/nfs-on-raspberry-pi-with-ufw-ntfs-and-connect-from-kodi/). 


# Helpful Links

- [NFS Raspberry Pi](https://www.raspberrypi.org/documentation/configuration/nfs.md)
- [Install NFS](https://vitux.com/install-nfs-server-and-client-on-ubuntu/)
- [NFS Subtree Check Info](https://www.suse.com/c/configuring-nfsv4-server-and-client-suse-linux-enterprise-server-10/#:~:text=no_subtree_check%20%E2%80%93%20If%20a%20subdirectory%20of%20a%20filesystem,is%20harder%29.%20This%20check%20is%20called%20the%20subtree_check.)


---

### Helpful links

- [How to configure a raspberry pi as PXE Boot Server](https://linuxconfig.org/how-to-configure-a-raspberry-pi-as-a-pxe-boot-server)
- [Automating Fedora Install w/ Kickstart](https://docs.fedoraproject.org/en-US/fedora/rawhide/install-guide/advanced/Kickstart_Installations/)
- [How to boot Fedora Rescue PXE](https://unix.stackexchange.com/questions/350813/how-to-pxe-boot-fedora-25-rescue)