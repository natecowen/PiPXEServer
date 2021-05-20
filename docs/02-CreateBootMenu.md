# Create A Boot Menu

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
                                append initrd=::boot/amd64/fedora/33/isolinux/initrd.img ip=dhcp inst.repo=nfs:192.168.x.x:/export/boot/amd64/fedora/33 inst.stage2=nfs:192.168.x.x:/export/boot/amd64/fedora/33 inst.ks=http://192.168.x.x/export/ks.cfg/ks.cfg



                MENU END
        MENU END

```

> The :: symbol above allows us to reference the tftp root in an absolute way. 

Ensure the current working directory is `/mnt/data/netboot` and run `ln -rs pxelinux.cfg bios && ln -rs pxelinux.cfg efi64`. This will link the menu configs for both bios and efi64 together. 

> From LinuxConfig.org article in Helpful Links Section: Here we used the -r option of the ln command to create relative symbolic links. 