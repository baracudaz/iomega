# Installing Debian on an Iomega StorCenter ix2-dl

These should be general instructions for getting normal Debian installed onto an Iomega StorCenter ix2-dl NAS

```bash
usb start
ext2load usb 0:1 0x00800000 /uImage
ext2load usb 0:1 0x01A00000 /uInitrd
setenv mainlineLinux 'yes'
setenv arcNumber '1682'
setenv bootargs 'console=ttyS0,115200 earlyprintk=serial'
bootm 0x00800000 0x01A00000
```

## Create images

```bash
cp -a zImage-5.2.9-kirkwood-tld-1 zImage.fdt
cat dts/kirkwood-lenovo-ix2-ng.dtb >> zImage.fdt 
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-5.2.9-kirkwood-tld-1 -d zImage.fdt  uImage
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs-5.2.9-kirkwood-tld-1 -d initrd.img-5.2.9-kirkwood-tld-1 uInitrd
```

## Working NAND memory

More info at [forum.doozan.com](https://forum.doozan.com/read.php?2,70294,70549#msg-70549) and also make sure to download files from [forum.doozan.com](https://forum.doozan.com/read.php?3,19216,48314#msg-48314)

```bash
cp zImage-5.4.2-kirkwood-tld-1 zImage.fdt
cat kirkwood-lenovo-ix2-dl.dtb >> zImage.fdt
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-5.4.2-kirkwood-tld-1 -d zImage.fdt uImage
```

## Boot

```bash
usb start
ext2load usb 0:1 0x00800000 /boot/uImage
ext2load usb 0:1 0x01A00000 /boot/uInitrd
setenv bootargs 'console=ttyS0,115200 root=LABEL=rootfs rootdelay=10 $mtdparts init=/bin/systemd'
bootm 0x00800000 0x01A00000
```

## Boot - new

```bash
setenv bootargs 'console=ttyS0,115200 root=LABEL=rootfs rootdelay=10 $(mtdparts) init=/bin/systemd'
setenv bootcmd_usb 'run mk_mtdparts; usb start; ext2load usb 0:1 0x00800000 /boot/uImage; ext2load usb 0:1 0x01A00000 /boot/uInitrd; bootm 0x00800000 0x01A00000'
run bootcmd_usb
```

## Boot - Final

```bash
setenv bootcmd_usb 'run mk_mtdparts; usb start; ext2load usb 0:1 0x00800000 /boot/uImage; ext2load usb 0:1 0x01A00000 /boot/uInitrd; setenv bootargs $(console) root=LABEL=rootfs rootdelay=10 $(mtdparts) init=/bin/systemd; bootm 0x00800000 0x01A00000'
run bootcmd_usb

setenv bootcmd_sata 'run mk_mtdparts; ext2load ide 0:1 0x00800000 /uImage; ext2load ide 0:1 0x01A00000 /uInitrd; setenv bootargs $(console) root=LABEL=ROOT rootdelay=10 $(mtdparts) init=/bin/systemd; bootm 0x00800000 0x01A00000'

setenv bootcmd_sata 'run mk_mtdparts; ext2load ide 0:1 0x00800000 /uImage.orig; ext2load ide 0:1 0x01A00000 /uInitrd.orig; setenv bootargs $(console) root=LABEL=ROOT rootdelay=10 $(mtdparts) init=/bin/systemd; bootm 0x00800000 0x01A00000'
run bootcmd_sata
```

## Base Setup

- [forum.doozan.com](https://forum.doozan.com/read.php?2,12096)
- [forum.doozan.com](https://forum.doozan.com/read.php?2,35331)
- [https://minaret.biz/tips/ix2/index.html#introduction]

## Swap

- [Linuxize: Add Swap Space on Debian 10](https://linuxize.com/post/how-to-add-swap-space-on-debian-10/)
- [DigitalOcean: Add Swap Space on Debian 10](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-debian-10)
