# Sun Netra X1
My notes about installing an OS onto a Sun Netra X1

## What is this?
Since the Sun Netra X1 doesn't have a video card or any video output, doesn't have a CD/DVD drive, doesn't boot from most IDE CDROM drives, and doesn't boot from the USB ports, it's actually quite a challenge to get an OS installed on this system. The two options are either sifting through boxes of CDROM drives until you find one that works, or try netboot. I chose the netboot option.

## What is netbooting?
Netbooting, also called PXE booting, is a way to load an OS over the network from the BIOS. The system loads the BIOS, initializes the network card, and sends requests to identify the network and retrieve a file from a server on the local network. Eventually the kernel is loaded and the OS is booted. This could either be a fully functional OS, a recovery image to restore the OS installed on the local disk, an OS installer, or just some temporary utility like a livecd, dban, clonezilla, gparted, etc. I'm going to be using an OS installer.

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
