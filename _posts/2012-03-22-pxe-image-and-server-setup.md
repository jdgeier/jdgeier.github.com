---
layout: post
title: "PXE Image and Server Setup for VMWare View"
category: ["linux"]
tags: ["linux", "PXE", "tftpd", "VMWare View"]
---
{% include JB/setup %}
## PART 1 building the initrd rootfs

All of the work shown in this documentation will be in the linux console. I will assume that the reader has a basic understanding of how the linux console works. Additionally you should know how to compile the linux kernel from scratch. If you do not know how to compile the linux kernel please refer to the link provided in the notes later on in the kernel section. It may be a good idea to familiarize yourself with the boot agent you use as well to ensure that the kernel is in working order.


#### Install prereqs

Bring up a console window. If you do not use a window manager just login. The first thing we will do is ensure that cramfs support has been installed and configured and boot strap a chroot environment for building our rootfs.

<pre class="prettyprint ">
apt-get install cramfsprogs
mkdir pxebuild
debootstrap etch pxebuild
vi pxebuild/etc/fstab
</pre>

#### Modify pxebuild/etc/fstab

Because we will be using cramfs, a compressed read only filesystem, certain entries need to be added to the fstab so that X and the vmware-viewer can create lock files. Neither DevFS or udev are used we will populate the device nodes with mknod when nessicary.

*pxebuild/etc/fstab*
<pre class="prettyprint linenums">
proc                /proc          proc    defaults                           0 0
tmpfs               /tmp           tmpfs   defaults,noatime                   0 0
tmpfs               /var/lock      tmpfs   defaults,noatime                   0 0
tmpfs               /var/log       tmpfs   defaults,noatime                   0 0
tmpfs               /var/run       tmpfs   defaults,noatime                   0 0
tmpfs               /var/tmp       tmpfs   defaults,noatime                   0 0
</pre>

#### Chroot to new environment

We can now chroot into our environment and begin to install items nessicary to making vmware-viewer functional.

<pre class="prettyprint ">
chroot pxebuild/
apt-get install xserver-xorg-video-i810
dpkg --purge xserver-xorg-input-evdev
dpkg --purge xserver-xorg-input-all
apt-get install xserver-xorg-input-kbd xserver-xorg-input-mouse
dpkg --purge xserver-xorg-input-evdev
dpkg --purge xserver-xorg-input-wacom
dpkg --purge xserver-xorg-input-synaptics
apt-get clean
dpkg-reconfigure -a
</pre>

#### Setup Network interfaces

The network interface needs to be setup to assign an ip address via dhcp in order to proceed, it is also nessicary to ensure that an entry is assigned for the loop back interface for vmware-viewer to work.

*/etc/network/interfaces* (in chroot env)
<pre class="prettyprint linenums">
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
</pre>

#### Modify inittab

Our goal here is to remove the console based login and prevent extraneous tty’s from spawning, this will prevent most common attempts that may be used to drop into a console via the function key combinations. Later we will add a line to rc.local to spawn the X instance with our client software. Just comment out the other lines.

*/etc/inittab*
<pre class="prettyprint linenums">
1:2345:respawn:/bin/bash
#2:23:respawn:/sbin/getty 38400 tty2
#3:23:respawn:/sbin/getty 38400 tty3
#4:23:respawn:/sbin/getty 38400 tty4
#5:23:respawn:/sbin/getty 38400 tty5
#6:23:respawn:/sbin/getty 38400 tty6
</pre>

#### Clean up the cruft and install client

Next we need to clean extra packages that have been installed automatically via the bootstrap command. This will save us some space later on when we compress the image. 

<pre class="prettyprint ">
dpkg --purge ppp pppconfig pppoe pppoeconf whiptail libnewt0.51 libpcap0.7
dpkg --purge logrotate libpopt0 at sysklogd klogd
dpkg --purge iptables ipchains telnet
dpkg --purge cron exim4 exim4-base exim4-config exim4-daemon-light mailx
dpkg --purge cron exim
dpkg --purge base-config adduser makedev
dpkg --purge gettext-base
dpkg --purge nano nvi ed
dpkg --purge man-db manpages groff-base info
dpkg --purge pciutils bsdmainutils fdutils cpio modutils
dpkg --purge console-tools console-common console-data libconsole
</pre>

Now we can add the dependancies needed to install our vmware client as well as download and install the client itself. 

<pre class="prettyprint ">
apt-get install gtk
apt-get install libxml
apt-get install gtk+
apt-get install rdesktop
wget http://vmware-view-open-client.googlecode.com/files/VMware-view-open-client_2.1.1-144835_i386.deb
dpkg --install VMware-view-open-client_2.1.1-144835_i386.deb
apt-get install libgtk2.0-0
rm VMware-view-open-client_2.1.1-144835_i386.deb
apt-get clean
</pre>

#### Setup X to spawn via rc.local

We will now cause X to spawn a single instance of the client software via the rc.local entry when the vmware client instance is terminated the machine will reboot itself. 

*/etc/rc.local*
<pre class="prettyprint linenums">
xinit -e vmware-view && reboot
exit 0
</pre>

#### Modify view-preferences

Since we are droping directly to a root shell without login the bash profile is not read and a home directory is not set this means that the vmware  client will read preference from /.vmware/view-preferences not from /root/ vi .vmware/view-preferences. We will need to modify the former as the latter will not be read. 

*/.vmware/view-preferences*
<pre class="prettyprint linenums">
.encoding = "UTF-8"
view.broker0 = "SERVERNAME"
view.defaultDomain = "DOMAINNAME"
</pre>

#### Clean up before exiting chroot

A little cleanup and we can exit the chroot environment for our build and.

<pre class="prettyprint ">
apt-get  clean
dpkg --purge apt apt-utils base-config aptitude tasksel
mknod /dev/input/mice c 13 63
exit
</pre>

## PART 2 Compiling the kernel and setting up tftpd.

### Compile the kernel
Note: I will not be writing a tutorial on how to compile the kernel. Unfortunately it would be very time consuming to describe. Therefore read [this tutorial](http://www.linuxplanet.com/linuxplanet/tutorials/3196/1/) if you do not know how to compile the kernel. If you have compiled a kernel before then proceed. The kernel that I have made is very minimal. Please be aware that excess size in the kernel will result in an increase in boot time. It is better the streamline the kernel by only using the modules that are needed for your hardware and then compile them directly into the kernel rather than as modules as the boot time is faster. 

<pre class="prettyprint ">
apt-get install linux-source-2.6.18
cd /usr/src/
tar xjvf linux-source-2.6.18.tar.bz2
ln -s linux-source-2.6.18 linux
cd linux
</pre>

It will be nessicary at this point to install your toolchain if you have not already done so. If you have already installed automake, gcc and binutils etc. skip this step. 

<pre class="prettyprint ">
apt-get install build-essential
apt-get install gcc
apt-get install libncurses5
apt-get install libncurses5-dev
</pre>

To compile the kernel use:

<pre class="prettyprint ">
make menuconfig
make && make bzImage
cp arch/i386/boot/bzImage ~
</pre>

### Install and configure tftpd

Now it is time to install and configure our tftpd deamon.

<pre class="prettyprint">
cd /var/lib/tftpboot/
mkdir pxelinux.cfg
touch default
apt-get install tftpd-hpa
</pre>

*/var/lib/tftpboot/pxelinux.cfg/default*
<pre class="prettyprint linenums">
defualt linux
timeout 1
prompt 1
serial 0 9600
display display.msg

label linux		
	kernel vmlinuz
	append initrd=initrd.img ramdisk_blocksize=4096 ramdisk=128000 root=/dev/ram0
</pre>

Note: At this point it may be a good idea to test your kernel. You will need to familiarize yourself with the bootloader that you use and create an entry to point to the new kernel. 

Next we will copy the kernel to our tftp directory.

<pre class="prettyprint">
cd /usr/src/linux
make install
cp /boot/vmlinuz-2.6.18 /var/lib/tftpboot/vmlinuz
</pre>

In order for the correct tftpd config to be read we will need to comment tftpd startup line in inetd.conf as it will be read prior to our init process is started leading to an improper startup of the deamon. Comment or remove the following line

*/etc/inetd.conf *
<pre class="prettyprint linenums">
#tftp           dgram   udp     wait    nobody  /usr/sbin/tcpd  /usr/sbin/in.tftpd /var/lib/tftpboot
</pre>

And modify the following config file via:

*/etc/default/tftpd-hpa*
<pre class="prettyprint linenums">
#Defaults for tftpd-hpa
RUN_DAEMON="yes"
OPTIONS="-l -s /var/lib/tftpboot"
</pre>

We should now have a properly configured tftpd running. All that is needed is to modify the scope on the DHCP server for the subnet that will be booting from the pxe image. 


