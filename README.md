# Sun Netra X1
My notes about installing an OS onto a Sun Netra X1

## What is this?
Since the Sun Netra X1 doesn't have a video card or any video output, doesn't have a CD/DVD drive, doesn't boot from most IDE CDROM drives, and doesn't boot from the USB ports, it's actually quite a challenge to get an OS installed on this system. The two options are either sifting through boxes of CDROM drives until you find one that works, or try netboot. I chose the netboot option.

## What is netbooting?
Netbooting, also called PXE booting, is a way to load an OS over the network from the BIOS. The system loads the BIOS, initializes the network card, and sends requests to identify the network and retrieve a file from a server on the local network. Eventually the kernel is loaded and the OS is booted. This could either be a fully functional OS, a recovery image to restore the OS installed on the local disk, an OS installer, or just some temporary utility like a livecd, dban, clonezilla, gparted, etc. I'm going to be using an OS installer.

# Phase 1: The Hardware
## Console output
No video output means you'll have to use the console port. Rather than use a PC with a serial DB9 port to directly connect to it, I built a portable terminal server using a Raspberry Pi Zero W, a USB OTG adapter, a USB-to-DB9 adapter, and finally a DB9 to RJ45 Cisco terminal server cable. 9600 8N1 later, and I can telnet into my Raspberry Pi like it's a real terminal server!

## The LOM (Lights Out Management)
If you connect to the first console port you're actually talking to (or through) the LOM. The hotkey `#.` will bring you to the LOM (with a `# ` prompt). It's always on if the server is connected to power, even if the server isn't "running".

Most useful LOM commands:
* `poweron`: Fires up the server and starts OpenBoot
* `poweroff`: Shuts down the server regardless of what state the OS is in
* `break`: Most of the time, get to an OpenBoot prompt. Sometimes goes back to the OS console. I don't know why
* `console`: Return to the OS console
* `reset`: Reset/Reboot the server
* `env`: Lists hardware data (fan speed, temperature, voltages, etc)
* `help`: See all LOM commands

## The Netra "BIOS" (OpenBoot)
Sun's OpenBoot firmware is actually quite versatile. Even when the OS is booted, you can switch to the OpenBoot prompt by hitting `#.` to get to the LOM, then `break` to get to the OpenBoot menu. Note that while here the OS is completely frozen (no ticks, timers, IO, etc) - this can often crash the OS if you stay here too long and then try to return to the os with `console`! I'll refer to OpenBoot as OBP from now on.

Most useful OBP commands:
* `boot <mode> <file and options>`: Boots the specified mode (`disk` or `net`) with the specified options (e.g. `-v`)
* `printenv`: Show OBP configuration. I mostly look for `boot-device` and `boot-file`
* `set-defaults`: Reset all OBP parameters to default
* `setenv <parameter> <value>`: Set a parameter to a value

By default the system will try to boot from the disk, and if no OS is found it will try to netboot. When troubleshooting netbooting it's usually helpful to change the default boot mode to start with `net`.

## Where should I be now?
You should have some way of accessing the LOM over the first console port. You understand how to switch between the LOM prompt, the OBP prompt, and the OS prompt. You'll also need an ethernet cable connected to the first network port. Note that since we'll be relying on a lot of layer 2 ARP broadcast traffic you can't do any fancy wireless bridging, just use a plain ethernet cable and try to be as "close" (in terms of network route) to your host server as possible.

# Phase 2: Netbooting
So what actually happens when the server netboots? You can either `boot net` to use ARP/RARP or `boot net:dhcp` to use DHCP. I didn't want to deal with reconfiguring my primary router (since it's the DHCP server), so I went with RARP. That seems to be the more native option, anyway.

## My server
For the following, I'm referring to the Netra as the **client** and my primary home server as the **server**. The server is a generic Ubuntu 16.04 x86-64 server. It's separated from the client with a standard network switch, with everything hardwired.

## The RARP daemon
This is the first part of netbooting that will be common among all OS types. The client needs to identify the network in order to move onto the next step.

Ubuntu carries a `rarpd` package that acts as a RARP server. It listens for broadcast RARP messages and cross references the request to the `/etc/ethers` file. If it finds the MAC address from the request in the file, it will respond with the IP address. That's how the client gets an IP address on the network without DHCP. Note that the client won't know anything about the subnet or netmask, so make sure you're on a standard network like 192.168.1.X.

The format of `/etc/ethers` is `<mac address> <ip address>`, e.g.

`00:03:ba:22:83:a8 192.168.1.15`

## The TFTP Server
This is the second (and last) part of netbooting that will be common among all OS types. After the client gains network access, the next thing it does is try to find a TFTP server using broadcast IP packets.

Ubuntu carries a `tftp-hpa` package that contains a TFTP server. Not much to configure here except the directory to store the TFTP files. Mine ended up being `/var/lib/tftpboot/`.

### What is the client asking for?
Wireshark will tell you. Since you know the IP address you can filter by the IP to see the TFTP requests. The client uses a hex encoding of the MAC address as the filename to look for, e.g. `C0B0932B`. This is what you'll need to name the file in the TFTP folder, since that's what it will copy over the network and start loading. Note that the Netra X1 can only load ~4MB at this stage due to how the processor is initialized by OBP. If it runs out of memory before the file is done downloading you'll get the `Fast Data Access MMU Miss` error and will need to reset.

## Where should I be now?
The Netra should be getting an IP and trying to copy the boot file from the TFTP server. You can troubleshoot this by just using a text file, or really anything as long as it has the right name. If the client fails to boot it'll just sit with an error until you reset it, but if it can't find the file it will keep trying to send broadcast requests without any error messages.

Your life will be a lot easier if all of the above configuration is dialed in/locked down so you don't have to worry about it. Save the configuration and get the daemons running in the background so they are out of the way - the only thing you need to worry about after this is swapping files in the tftpboot folder.

This is where things start to deviate depending on the OS you're trying to install.

# Windows 10
Just kidding :)

# Debian
Sadly Debian dropped SPARC64 support after 7.0 (Wheezy), so that's the newest official version to get. I tried looking for Debian 10 images (the latest at the time of this writing), but couldn't find netboot images that were small enough for the Netra to load. It seems like the 4.X kernels are too large for the Netra to netboot :(

The official Debian 7 SPARC netboot image [can be found here](http://archive.debian.org/debian/dists/wheezy/main/installer-sparc/current/images/netboot/boot.img). Download this .img file into your tftp folder and let the Netra download it.

Once the tftp transfer is done, the Debian kernel and installer should start loading! Let it autodetect the network, then when it asks what mirror to use for the install scroll to the very top where it says **enter information manually**. The Debian archive mirror hostname should be `archive.debian.org`, and the Debian archive mirror directory should remain the default of `/debian/`. That's it!

## Installation notes
* The installer won't be able to access the Debian security repository. This is expected since they aren't publishing security updates for SPARC64 platforms.
* The clock in my Netra ran a little slow, preventing `ntpd` from maintaining a stable time sync. To fix this, stop `ntpd` and/or any other time sync daemons, then install `adjtimex`. During installation it will calibrate the clock and fix the tickrate. In my case, it adjusted the tickrate from 10000 to 10010. This can be seen with `adjtimex -p`.
* The silo bootloader works fine, but if you want grub you can install it with:
  * sudo apt-get purge silo
  * sudo apt-get install grub2
  * sudo grub-install /dev/sda
  * sudo update-grub

# Solaris
Solaris 11 dropped support for the UltraSPARC II series, so Solaris 10 is the newest version that can be installed.

Remember when I said you only need to worry about swapping files in the tftpboot folder from now on? Well Sun (now Oracle) likes to make things complicated, so there's more infrastructure to set up before anything will load.

## The ISO
You'll need to download and mount the Solaris ISO on the server. Grab the latest [SPARC version here](https://www.oracle.com/solaris/solaris10/downloads/solaris10-get-jsp-downloads.html), then mount it using a loopback. In my case, I mounted it like this: `/root/sol-10-u11-ga-sparc-dvd.iso on /mnt/iso type iso9660 (ro,relatime)`. Not sure if this matters, but everything is owned by `root` here.

## The TFTP Server
The file to be netbooted is actually hosted on the ISO. Copy `/mnt/iso/Solaris_10/Tools/Boot/platform/sun4u/inetboot` to `/var/lib/tftpboot` (or wherever your TFTP folder is) and rename it to the filename the client is looking for.

## Bootparams
The client will start netbooting at this point, but the file it downloads is just the first stage of the boot process. When this loads, it tries to download the full kernel and installer via an NFSv4 mount. It determines where this mount is with a bootparams request over the network.

Ubuntu carries a `bootparamd` package that contains a bootparams server. Install it and configure the `/etc/bootparams` file like this:

```netra root=192.168.1.3:/iso/Solaris_10/Tools/Boot \
                install=192.168.1.3:/iso/ \
                boottype=:in \
                rootopts=192.168.1.3:rsize=8192```

This was **EXTREMELY** finnicky, so make sure to copy this exactly if you're mounting things the exact same way as me. Otherwise paths will need to be tweaked. I followed [this guide](http://hintshop.ludvig.co.nz/show/solaris-jumpstart-linux-server/) to build my bootparams file - this might help you as well.

Note that `netra` was the hostname defined in my router at this point in time, meaning all DHCP requests from this MAC address automatically assigned a reserved IP and tied it to that hostname. I'm assuming an IP address would work here as well, but I haven't tested it.

## NFS
The NFS server is the final step to the boot process. This contains the mounted ISO which the booted client can pull from and use to install the OS.

Ignore what you read on the Internet - this can definitely work with NFSv4, if you configure it correctly!

Ubuntu carries a `nfs-kernel-server` package that contains an NFSv4 server. Install it and configure the `/etc/exports` file with this line:

`/mnt *(fsid=0,ro,async,no_root_squash,anonuid=0,anongid=0,no_subtree_check,crossmnt)`

This hosts the folders in `/mnt` (including `/mnt/iso` and others) to all IP addresses. This is why the `/etc/bootparams` file uses `/iso/` as the root path, because NFSv4 hides the mount point when exported. The *particularly* important parameter here is `crossmnt`, which makes mounted filesystems visible in the NFSv4 mount. This one took me a while to figure out, but the easy way to troubleshoot this is to try mounting this NFSv4 mount somewhere locally, and then browse into it and see if you can see anything.

## The Boot Process
In OpenBoot you'll want to boot with `boot net -v - install`. This gives you more verbose output of the boot process and kicks off the Solaris installer.

## Finally Installing
During install it will ask you a few questions. Note that if you're using a device that doesn't have function keys (like a chromebook), you can use the key sequence `Esc-number key` to emulate function keys during the install. For example, most of the install screens require pressing F2 to continue, but `Esc-2` also works. Once you hit `Escape` you should see the navigation bar at the bottom change to using number keys instead of function keys. You'll still have to hit `Escape` before every number, however.

### What type of terminal are you using?
I wasn't sure since I'm using a homemade Raspberry Pi terminal server, but I had the best luck with `12) X Terminal Emulator (xterms)`.

### Default Route for dmfe0
`Detect one upon reboot` doesn't seem to detect my default route upon reboot. I could select `Specify one` and manually enter `192.168.1.1`, or if I chose `Detect one upon reboot` during the install I just needed to use the `route` command to manually specify one once the OS was installed.

### Oracle registration
I don't know, I just entered my email address without any other account information.

### Select Software
`Entire Distribution plus OEM support`: This seems to produce error messages at the end of the install, causing the installer to fail.
`Entire Distribution`: This seems to hang at the very end of the install without any errors, but also without a bootable system.
`Developer System Support`: This was the largest software set I could get to install.
`Core System Support`: This installs nice and fast (relatively speaking), but lacked many of the system management tools I was expecting to have preinstalled. It's not worth the headache of manually installing packages off the DVD image from the Solaris command line.

## Installation notes
* Enable root login over ssh if you don't care about security: set `PermitRootLogin` to `yes` in `/etc/ssh/sshd_config` and restart sshd.
* The stock package manager, `smpatch`, won't work unless you have an Oracle Support account. This costs money, so I don't have that. Luckily, OpenCSW replaces this and is free (and awesome!)
* Did I mention OpenCSW is awesome? Here's some quick tips:
  * Install OpenCSW: `pkgadd -d http://get.opencsw.org/now`
  * Update package lists: `/opt/csw/bin/pkgutil -U -u -y`
  * Fix paths. Edit /etc/default/login, uncomment `PATH` and `SUPATH` and add `/opt/csw/bin:` to the start. In bash, run `export MANPATH=/opt/csw/share/man:/usr/share/man` to get man pages for the new packages.
  * Install vim: `/opt/csw/bin/pkgutil -i vim -y` and whatever else you want!
* Oddly the `Developer System Support` software set doesn't come with the native Solaris compiler. It does come with GCC, but I wanted to compare the GCC port with the native Solaris compiler so I had to find it elsewhere. It's part of the Sun (Oracle) Developer Studio software set, which can be [downloaded here](https://www.oracle.com/tools/developerstudio/downloads/developer-studio-jsp.html).
  * WARNING: Developer Studio 12.6 isn't compatible with UltraSPARC IIe, although it will install regardless and break a bunch of packages, including java. If you make the same mistake as me, and `java -version` no longer works, use `patchrm 119963-35` to uninstall the offending patch. I found this patch by looking at `showrev -p | grep -i libc` and uninstalling the newest patch that was found (since `java -version` was complaining that libCrun.so.1 was incompatible). Developer Studio 12.5 works with the UltraSPARC IIe.

# Gentoo
I can't get Gentoo to install. The [most recent experimental tftpboot image I found](https://gentoo.osuosl.org/experimental/sparc/tftpboot/sparc64/gentoo-sparc64-20100413.tftpboot) is from 2010 and the [most recent stage3 tarball it can extract](http://distfiles.gentoo.org/releases/sparc/autobuilds/20141201/multilib/stage3-sparc64-multilib-20141201.tar.bz2) is from 2014 (the `tar` executable provided by the tftpboot image only knows how to extract \*.bz2 files, not \*.xz files), and the tar command fails partway through extraction. I'm not a Gentoo expert, so I'm not sure if I'm doing something wrong or if it's a bad combination of tftpboot image and stage3 tarball.

# OpenBSD
Unfortunately I ran out of mental capacity before I could try this. From what I could tell the SPARC architecture is well supported by OpenBSD, even to the point where they provide the LOM(lite?) application in the OS, just like Solaris.
