---
layout: narrative
title: "A Virtualized Home Desktop/Network Setup Based on KVM"
author: Michael Shullick
rights: Public Domain
publication-date: 2020
toc:
- Goal
- Audience
- Hardware
- Requirements
- Host Configuration
- KVM Setup, Networking, and pfSense
- Home Assistant
- Windows
- Conclusion
---

## GOAL

I have a single machine that needs to do a lot of stuff. It’s gotta enable me to play games with my close friends in far away places, it’s gotta be my router/firewall, and it’s gotta manage my smart devices in my home. And yet, I don’t want 3 different machines to do these things.

So I says to Linux, “You get one box.” And it says, “ok”.

My solution is to run an Arch Linux host system, and run virtualized instances of pfSense, Home Assistant, and Windows, each capable of near native-level security and performance.

## AUDIENCE

I assume the reader has at least a moderate level of experience with Linux, and is familiar with basic operations on Arch Linux, such as package installation, distro conventions, and other common tasks. Additionally, I assume the reader knows which commands are appropriate to run as root.

## HARDWARE

- Graphics cards: 1x GTX 760 (for use by host) and 1x GTX 1070 (for use by Windows guest)
- Displays: Two monitors, both with 2x HDMI and 1x Displayport. I connected both monitors to both cards, such that I could switch inputs on the monitors and see both the Host xfce desktop and the Windows desktop.
- Network topology: Modem --> Mobo, Mobo --> Switch

## REQUIREMENTS

- All OSs can be run at once, and hardware delegation works swimmingly
- My WAN port is configured for pci-passthrough to pfSense, and the LAN port is bridged between my KVM guests. pfSense provides DHCP on the LAN.
- Windows must also have exclusive hardware access to the better video card, which must run my monitors at 4K @ 60Hz.


## HOST CONFIGURATION

Simple stuff here. I plan to wipe my hard disk for a clean slate and then run all of my VMs out of `raw` hard disk images as needed. We’ll revisit that later as needed.

So on to installing a clean slate of Arch. Of course, it goes without saying--back up your sh\*t.

### Installation
I’m making a new empty GPT partition table on /dev/sda, and making a new EFI system partition with 280MB per Arch Wiki recommendation. [\[link\]](https://wiki.archlinux.org/index.php/EFI_system_partition#Create_the_partition)

I filled up the rest of the disk with an ext4 partition. Opting for no swap space because I have 32GB of memory available. Adjust this as appropriate for your machine.

Now run the rest of the installation [as documented](https://wiki.archlinux.org/index.php/Installation_guide).

 For the bootloader, I opted to use EFISTUB by following the directions [here](https://wiki.archlinux.org/index.php/EFISTUB), without the swap/hibernate stuff.

Because I ended up deleting and recreating the boot entry _a lot_ as part of figuring out kernel parameters, I scripted the entry. For convenience, this is the final version of the script:

~~~ bash
#!/usr/bin/env bash
efibootmgr -B -b 0000 # Delete the old primary entry
efibootmgr --disk /dev/sdc --part 1 --create \
    --label "shoecore - Arch Linux" \
    --load /vmlinuz-linux --unicode \
    'root=PARTUUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX \
    rw initrd=\initramfs-linux.img quiet amd_iommu=on iommu=pt \
    vfio-pci.ids=XXXX:XXXX,XXXX:XXXX acpi_enforce_resources=lax' \
    --verbose
~~~
Some notes on this script:
- You will need to change the partition UUID to match your system partition.
- You will need to change `amd_iommu` to `intel_iommu` if you're using an intel-based system
- I will touch on the other kernel parameters later on in this post. You can probably omit them as needed if you need to isolate problems with booting your host system.

After creating this entry, you should have a virgin Arch Linux install that boots without issues.

### Basic Configuration
First thing I did was install `nvim`, just so I had a go-to editor ready to go. From this point, I was running everything as the root user for convenience until I specified otherwise below.

For initial networking, I ran a manual set set of `ip` commands until I had pfSense figured out. Reminder you can see your nic names with `ip link`.

My motherboard has two ethernet ports. When you run `ip link`, it may not be immediately obvious which link corresponds to which physical port on your motherboard. I denoted the physical ports by referring to the "upper" or "lower" one. I decided at this point the lower one would be my WAN port, and the upper for use on my LAN. To identify which was which, I followed the following procedure.
1. Add untaken static addresses to both:
```
ip addr add 192.168.1.200/24 dev enp_we1
ip addr add 192.168.1.201/24 dev emp_we2
```
(“weX” referring to “whatever” 1 and 2 because I don’t remember the default names)

2. Connect a cable between your router and one of the ports--say the lower one.
3. Bring up one link at a time and see which enables us to ping the router.
```
 ip link set enp_we1 up
 ip route add default via 192.168.1.1
 ping 192.168.1.1
```
If you get a response, then we know `enp_we1` is the lower one. If not, you can probably conclude `enp_we2` is the lower. But I'm an idiot, so I want to see it for myself. To try the other interface, first we have to remove the route on the first. If you run `ip route` you’ll see the default route was tied to the interface.
```
> ip route
default via 192.168.1.1 dev enp_we1
192.168.1.0/24 dev eth_lower proto kernel scope link src 192.168.1.201
```

Delete the default route tied to the interface: `ip route delete default`

Bring down the first interface: `ip link set enp_we1 down`

Bring up the other one: `ip link set enp_we2 up`

Add the default route again `ip route add default via 192.168.1.1`

At this point I was able to ping my router successfully. I then added a DNS server so I could update my system. `echo “nameserver 1.1.1.1” >> /etc/resolv.conf`.

At this point, I changed my network interface names so I didn’t have to keep track of which physical ethernet port was which ugly name. The procedure for this is documented [on the Arch Wiki](https://wiki.archlinux.org/index.php/Network_configuration#Change_interface_name).

Here, I also set up a user account with `useradd`, and installed `sudo`. From now on everything is done as my user account and not root.

Additionally, I realized getting virtualized displays was going to be 10x easier with a desktop environment, so I decided last minute to install xfce. I installed the package `xfce4` with options 9 10 11 12 13 14 (they are space delimited at the command line). I installed all options for xorg because I didn't care to figure which I didn't need. I also opted to use xinit instead of session manager because I don't want my host system to use resources on a desktop environment unnecessarily.
```
pacman -S xfce4 xorg xfce4-goodies xorg-xinit
```

Create an `.xinitrc` in your user folder. I just copied the default and replaced the lines that start the twm stuff to start `xfce4` and `xfce4-panel`. At this point I was just able to run `startx` and got an excessively high resolution xfce desktop. Perfect.

Now we're ready to dive into KVM.

## KVM SETUP, NETWORKING, AND PFSENSE

Start here with ensuring your CPU and Linux install are capable of this kind of virtualization. [\[Link\]](https://wiki.archlinux.org/index.php/KVM#Checking_support_for_KVM)

Then, install `qemu` and `libvirt` base packages. Per the libvirt page, I also installed these:
```
pacman -S qemu libvirt ebtables dnsmasq bridge-utils openbsd-netcat virt-manager
```

Start and enable libvirtd:
```
Sudo systemctl start libvirtd
Sudo systemctl enable libvirtd
```

Then we need to enable PCI passthrough so pfSense can have direct, exclusive access to the WAN port. You need the kernel flags specified here (already included in the provided boot entry script above). \[[Link\]](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_IOMMU)

The bits about IOMMU groups and isolated slots are only relevant for GPU passthrough, which isn't needed. Right now we only care about the kernel flags. So in short, if you used my script above, there's nothing to do for this step.

Download the pfsense image from [here](https://www.pfsense.org/download/).
With xfce installed you can just install firefox as well and download it.
```
https://nyifiles.pfsense.org/mirror/downloads/pfSense-CE-2.4.5-RELEASE-p1-amd64.iso.gz
https://nyifiles.pfsense.org/mirror/downloads/pfSense-CE-2.4.5-RELEASE-p1-amd64.iso.gz.sha256
```

Took about a minute, then validate it:
```
sha256sum <pfsense.gz> | sha256sum -c <path to pfsense.gz.sha256>
```

Should say “OK”. Then, decompress: `gunzip <pfsense.gz>`

Before we create the pfSense VM, we need to set up the bridged network for our LAN so the right option can be specified in `virt-manager`. For this, we will use `netctl`[^fn1].

Install netctl: `pacman -S netctl`

I started with configuring the bridge with netctl per [this page](https://wiki.archlinux.org/index.php/Bridge_with_netctl). Something didn't work in the first go, but this configuration, (`/etc/netctl/br0`) has proven reliable:

```
Description="Virtualized Bridge Interface"
Interface=br0
Connection=bridge
BindsToInterfaces=(eth_upper)
## Use the MAC address of eth1 (may also be set to a literal MAC address)
MACAddress=eth_upper
IP=static
Address='192.168.1.2/24'
Routes=('default via 192.168.1.1')
## Ignore (R)STP and immediately activate the bridge
SkipForwardingDelay=yes
```

[^fn1]: This is perhaps where I spent the most time debugging: the network bridge. Unfortunately, I got caught up in troubleshooting, so I didn't write down everything I tried. Ultimately, I ended up setting up a network bridge using `netctl`. This was actually the first thing I tried, but for some reason, it didn't work. I got stuck being unable to ping pfSense from my host OS. I tried many things that did *not* work, including: 1) Passing through both NICs (wouldn't make any sense, since other guests wouldn't have access) 2) Using a `macvtap`. Didn't work, I don't think `libvirt` would even start the VM 3) Using a bridge created by `Network Manager`, which worked for a day and then randomly stopped working. 4) Screwing with arbitrary routing rules (wouldn't make sense, since a bridge is lower on the stack than routing) 5) Instructions on [this page](https://turlucode.com/qemu-kvm-bridged-networking/), [this page](https://kyau.net/wiki/ArchLinux:KVM#Network_Bridge), or the stuff with using `qemu-bridge-helper` on [this page](https://wiki.archlinux.org/index.php/QEMU#Running_virtualized_system). The directions on [this page](https://jamielinux.com/docs/libvirt-networking-handbook/) alone did not get everything working, but the explanation of the concepts was very helpful.

Now we create our virtual machine. At this point, I used the GUI interface `virt-manager` to set up my VM. The steps were:
1. I specified the (decompressed) image file I downloaded
2. I specified that the OS was FreeBSD 11.3 based on [this page](https://docs.netgate.com/pfsense/en/latest/releases/versions-of-pfsense-and-freebsd.html) for the version of pfSense I downloaded.
3. I gave it 1CPU, 1GB of ram, and 2GB of disk space based per the [requirements](https://www.pfsense.org/products/#requirements).
4. I named the disk file just plain old “pfSense”
5. I checked the box for “Customize configuration before install”
6. In the configuration window, I hit “Add Hardware” and select “PCI host device” and chose the device which was labeled with my `eth_lower`

    ![KVM pfSense WAN eth passthrough virt-manager config](/assets/kvm_pfsense_passthrough_nic.png)

7. For the other NIC, the working bridged/tap configuration looked like this:

    ![KVM pfSense LAN Bridge virt-manager config](/assets/kvm_pfsense_lan_bridge.png)

8. I also checked the box to enable pfSense to start automatically every time my computer boots.
    ![KVM pfSense boot on boot](/assets/kvm_boot_on_boot_virt_manager.png)
9. I also had to switch from the default Spice display to VNC in order to boot without a running X desktop. Select the Display Spice section, and change the "Type" to VNC server.

As an aside, after I switched to the final wiring with the shared LAN nic and put the WAN nic directly behind my modem, I had issues with pfSense getting the WAN address and getting the basic internet connection working. I ended up making a fresh pfSense instance in kvm, rebooted my modem, and restarted the _new_ pfsense once post-install, and it all suddenly started working[^fn2].

[^fn2]: To help troubleshoot bridging, I simplified the set up a bit. I jumped ahead and created a Windows VM and gave its NIC a static IP. I tested various bridging configurations by trying to ping Windows from my host Arch install. When this worked, it proved there were no issues with bridging. It was a bit hairy testing this type of thing with pfSense because the LAN port on pfSense was supposed to run DHCP, and I couldn't play with various settings since I was unable to reach the web UI without a working network configuration.

I couldn't tell you exactly how the stars aligned, but all the pieces started working. From here on, this document was written in Google docs behind a virtualized instance of pfSense (whoo!).



## Home Assistant

This was very simple using the directions provided on [Home Assistant's site](https://www.home-assistant.io/hassio/installation/). I downloaded the QCOW2 file and ran through the directions. I also had to switch to the VNC display as was done with pfSense in order to boot headlessly.

The only gotcha was that apparently KVM does not have a built in mechanism to enforce the order in which your VMs boot if you just use the default "Start virtual machine on host boot up" as I did for pfSense above. This mattered because if Home Assistant and pfSense boot at the same time, it seems Home Assistant fails to get an address by DHCP, and it never becomes available. To work around this, I made a one-shot `systemd` service to start Home Assistant 30 seconds after the libvirtd daemon starts and boots pfSense. Here's `/etc/systemd/system/vmstart.service`:

```ini
[Unit]
Description=Non-pfSenseVMs
Requires=libvirtd.service

[Service]
Type=oneshot
ExecStart=sleep 30
ExecStart=virsh start HomeAssistant

[Install]
WantedBy=multi-user.target
```

Once that's saved, run `systemctl enable vmstart`. Now Home Assistant should be ready every time your computer reboots shortly after pfSense becomes available. You can add additional lines to start other VMs as needed with this same service.


## Windows

Windows worked shocklingly easily (once you know what you're doing). A general look-over of [this page](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF) is recommended.

Before we dive into creating the VM, it's import to understand how the PCI passthrough works for the graphics cards, because it's trickier than the ethernet. This is complicated, and I'm going to try to explain it in a nut shell. As reminder, my intended set up is to have one graphics card available to my host OS, and another beefier one passed through to Windows.

To begin, when your computer boots, you need to configure your machine to not start using the video output on the card which you want to pass through. At all. It needs to be completely isolated so your guest has full control. There can be no "sharing".

There are two parts to this. First, while booting, the Linux kernel starts streaming logging output to the default output specified by your board's firmware. Some motherboards offer an option for this:

![intial vid output](/assets/kvm_win10_pci_initial_output.png)

My first motherboard had this option once I updated the BIOS. I was able to specify the built-in IGFX for use by the host OS, which left the PCI card open to passthrough to Windows. Halfway through the project, however, I upgraded some hardware and my new motherboard did _not_ have this option, and to boot, my new CPU did not have integrated graphics. This is where I was fortunate to have two video cards while only needing to passthrough one of them. I worked around not having the option to choose a default video output in the BIOS by physically putting the card I did _not_ want to passthrough in whichever slot the BIOS used by default to display the splash screen, configuration menu, etc.

The second thing you have to do is tell Linux not to initialize the default driver for the passthrough card. Otherwise, it will start trying to use the card and that's no bueno. To get around this, the KVM people have provided a stub driver which satiates the kernel but prevents it from beginning to use the card. For this step, [this section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Isolating_the_GPU) of the Arch Wiki has solid instructions.

In the EFISTUB script I provided at the beginning, you can see where I specified device IDs that should be used by vfio-pci. Pay special attention to the Gotcha section in the "Setting up IOMMU". Specifically, when specifying the IDs from your GPUs IOMMU group, make sure to exclude the PCI root port. You should only specify the IDs of the VGA card and the audio device.

As an aside, *If you don't have two cards*, I _believe_ what you can do is the following:
1. Ignore all the stuff I said above about setting the default graphics card in your BIOS
3. Make sure Linux is set up to be automatically accessible over the network (such as by SSH).
2. Configure your Linux install to use the stub driver on your PCI card
4. Add the kernel option `quiet` (already set in the script I provided earlier in the post). This way Linux won't begin outputting dmesg output while booting, and the PCI card will still be available for passthrough.

I haven't tested the above steps; there may be other options necessary to fully disable video output by the kernel, but this should enable you to boot Windows either automatically like our other VMs, or by manually `ssh`ing in and issuing the command to boot the Windows guest.

But anyway, with the GPU you intend to passthrough isolated, and your host booted and ready to receive commands (either by a desktop available through a second graphics card, or over the network with SSH or XForwarding), we can proceed with creating our Windows VM. Given my set up, I still did this all through the `virt-manager` GUI with my secondary card.

1. Download an image of Windows 10 [here](https://www.microsoft.com/en-us/software-download/windows10ISO).
2. Create a new VM with `virt-manager`, and choose to boot local install media. Select the path to your downloaded Windows ISO.
3. Choose your desired amount of memory, but *only choose 1 CPU for now*. This doesn't refer number of cores to allocated to Windows, it refers to the number of virtual _CPU sockets_ to allocate. Depending on which version of Windows you choose to install, you may be capped at 1 CPU. We'll allocate multiple CPU _cores_ later, but leave this setting at 1 CPU.
4. Create storage for the VM. The `raw` format is recommended for performance. You can find more about this here.
5. Make sure you hit "Customize configuration before install" before hitting finish.
6. In the "Overview" section, make sure you choose "i440FX" for your chipset and "UEFI x86\_64: /usr/share/edk2-ovmf/x64/OVMF\_CODE.fd" for the firmware. I struggled to get the system to boot without this. I did _not_ try the secure boot option.
7. Under "CPUs", this selection below worked well for me. You can decide how many cores are appropriate. I didn't experiment with this much.
    ![Win10 CPUs](/assets/kvm_wind10_cpus.png)
8. Delete any virtual display devices. Since we intend to display only through the passed through card, we don't want to confuse Windows.
9. For networking, it should be configured just like Home Assistant--another tap on the bridge we created.
10. USB devices (Mouse, keyboard, joystick...) must be passed through to control Windows. Bear in mind whatever you passthrough will not be available to the host until the guest is shut down. At the time of writing, there's a bug that prevents you from specifying USB passthrough via the GUI. You must do it via `virsh`, the CLI equivalent of `virt-manager`.  [\[Directions\]](https://david.wragg.org/blog/2009/03/using-usb-pass-through-under-libvirt.html)
11. Finally, attempt booting.

Ideally, you'll get a very low resolution Windows output on the display attached to your passed-through card. Go through all of the installation according to your needs, and when you finally get to the desktop, install the driver for your graphics card.

Once that's set up, you should have a normally functioning Windows installation. I've found that "near-native performance is accurate, with Microsoft Flight Simulator 2020 working quite well on a passed-through Nvidia GTX 1070.

## Extras

That extra kernel option, `acpi_enforce_resources=lax`, was needed for me to see temperatures with my particular motherboard. [\[Reference\]](https://wiki.archlinux.org/index.php/Lm_sensors#Asus_B450/X399/X470/A320M-K/A320M-K-BR_motherboards_with_AM4_Socket)

## Conclusion

At the end of the day, I've met all of my requirements with this set up. There's a number of things I still want to set up (e.g., DNS, another guest with Pi-hole, a VPN via pfSense...), but this is been a good proof of concept.

I've found that at idle, with pfSense and home assistant running, the system pulls about 100 watts.

I think it's still ideal to have discrete machines for all these things, because all of the guests and host are now coupled such that if the host goes down, I lose my whole network, but for someone in my point in life this works very well.


