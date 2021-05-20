# Configure Network Items

Overview: This section covers DNSMasq, Firewall Settings, and Network File Share

---

## Configure DNSMasq

Normally, PXE servers will act as a DNS Server. This will cause problems if there is another DNS server on the network. I use my router to act as a DNS server. I also assigned a static IP to the PI PXE Server in my routers DNS/DHCP settings. 

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

The Network File System is used to serve linux files during installation to the Intel NUCs. 

Normally, it would be used for also loading the Kickstart file. However, there is a bug with NFS and Redhat/Fedora/CentOS installations where it will double mount the file share. This will throw an error similar to `dracut-initqueue: cp: cannot stat '/run/install/ks.cfg.` To get around this... [Redhat suggest](https://access.redhat.com/solutions/3938501) using a second NFS Server or an HTTP Server to send the kickstart file. *Note: I have left the kickstart fstab/export settings below in case this is fixed in the future.*

```shell
# install
sudo apt install nfs-kernel-server

# Make Export Directory
sudo mkdir -p /export/boot
sudo mkdir -p /export/ks.cfg

# Set permissions
sudo chmod -R 0777 /export/boot
sudo chmod -R 0777 /export/ks.cfg

# Bind Mount the Directory
sudo nano /etc/fstab

# add the following to fstab to ensure mounts work after reboot and save
/mnt/data/netboot/boot    /export/boot   none    bind  0  0
/mnt/data/netboot/ks.cfg /export/ks.cfg  none    bind  0  0


# Mount and verify fstab mount works
sudo mount -a

# Verify mount via df
df -h /export/boot


# add our directories to local network: 
sudo nano /etc/exports

# add and save the following lines (you can do a whole subnet via something like 192.168.100.0)
/export       192.168.2.0/24(rw,nohide,insecure,no_subtree_check,async)
/export/boot 192.168.2.0/24(rw,nohide,insecure,no_subtree_check,async)
/export/boot/amd64/fedora/33 192.168.2.0/24(rw,nohide,insecure,no_subtree_check,async)
/export/ks.cfg 192.168.2.0/24(rw,nohide,insecure,no_subtree_check,async)



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

---

## Install Apache Server

As stated above, there is a bug with installing Kickstarted files via NFS. To get around this you can serve the kickstart file via Apache.

```shell
# install apache2
sudo apt install apache2 -y

# check apache 2 listing in UFW
sudo ufw app list

# if apache is not listed edit application.d
sudo nano /etc/ufw/applications.d/apache2-utils.ufw.profile

# copy, paste, and save the following
[Apache]
title=Web Server
description=Apache v2 is the next generation of the omnipresent Apache web server.
ports=80/tcp

[Apache Secure]
title=Web Server (HTTPS)
description=Apache v2 is the next generation of the omnipresent Apache web server.
ports=443/tcp

[Apache Full]
title=Web Server (HTTP,HTTPS)
description=Apache v2 is the next generation of the omnipresent Apache web server.
ports=80,443/tcp

# reload ufw
sudo ufw app update appname

# check apache 2 listing in UFW
sudo ufw app list

# add Apache UFW Settings like this
sudo ufw allow from 192.168.x.0/24 to any app apache

```

## Set Apache to Serve Kickstartfile

```shell
cd /etc/apache2/sites-available 

sudo nano 000-default.conf

# Add the following line - alias & directory
ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html      
        Alias   "/export/ks.cfg" "/export/ks.cfg"

        <Directory "/export/ks.cfg">
                Require all granted
        </Directory>


# reload apache
sudo service apache2 reload

# In a web browser, verify you can load: 
http://192.168.X.X/export/ks.cfg/ks.cfg

```

---

## Helpful Links

- [NFS Raspberry Pi](https://www.raspberrypi.org/documentation/configuration/nfs.md)
- [Install NFS](https://vitux.com/install-nfs-server-and-client-on-ubuntu/)
- [NFS Subtree Check Info](https://www.suse.com/c/configuring-nfsv4-server-and-client-suse-linux-enterprise-server-10/#:~:text=no_subtree_check%20%E2%80%93%20If%20a%20subdirectory%20of%20a%20filesystem,is%20harder%29.%20This%20check%20is%20called%20the%20subtree_check.)