# Homelab

This repository serves as a guide for the creation of a personal homelab, with the objective of learning and testing technologies. The points defined here should not necessarily be followed without modifications, since it only meets my personal needs. Anyway it should still serve as a good starting point and inspiration.

## Hardware
 - Internet 600Mbps
 - Mini PC Intel Core i7-10700 16GB 512GB SSD with 2 NICs 1GbE
 - Switch TP-Link TL-SG1210P
 - Access Point TP-Link EAP660 HD

## Installation
### Proxmox
Before we begin our infrastructure looks like this:

![proxmox_0](static/proxmox/proxmox_0.png)

In my case the ISP Router is using 192.168.1.1/24 for the DHCP server. I will connect the interface enp2s0 to the router and use the IP 192.168.1.100 for the Proxmox server. We will change this IP later to be served by the internal OPNsense server.

The first thing necessary is to configure the BIOS of the machine to enable virtualization. In my case, I had to enable the following options:
 - Intel Virtualization Technology
 - Intel VT-d
 - Intel VT-x

After that, we can proceed with the installation of Proxmox. To do this, we need to download the ISO from the [official website](https://www.proxmox.com/en/downloads/category/iso-images-pve) and burn it to a USB stick. Then we need to boot from the USB stick and follow the installation steps.

This is the screen when you first boot from the USB stick. Choose `Install Proxmox VE (Graphical)`:

![proxmox_1](static/proxmox/proxmox_1.jpg)

Click to accept the license agreement:

![proxmox_2](static/proxmox/proxmox_2.jpg)

Choose where to install the server, in my case I will choose the entire nvme disk with the ext4 file system:

![proxmox_3](static/proxmox/proxmox_3.jpg)

Choose the country, time zone and keyboard layout:

![proxmox_4](static/proxmox/proxmox_4.jpg)

Choose the password for the root user:

![proxmox_5](static/proxmox/proxmox_5.jpg)

Choose the NIC interface, hostname, IP address, gateway and DNS server. In my case this will be:
 - Interface: enp2s0
 - Hostname: proxmox.mfurlanetto.com
 - IP: 192.168.1.100/24
 - Gateway: 192.168.1.1
 - DNS: 1.1.1.1

![proxmox_6](static/proxmox/proxmox_6.jpg)

Click in `Install` to start the installation.

![proxmox_7](static/proxmox/proxmox_7.jpg)

The installation will take a few minutes:

![proxmox_8](static/proxmox/proxmox_8.jpg)

When the installation is finished, remove the USB stick and reboot the machine.

When the machine boots, you will see the following screen with the URL to access the server:

![proxmox_9](static/proxmox/proxmox_9.jpg)

After acessing the URL you will probably see a warning about the self signed certificate, click to proceed anyway:

![proxmox_10](static/proxmox/proxmox_10.png)

Login with the root user and the password you defined during the installation:

![proxmox_11](static/proxmox/proxmox_11.png)

After login you will see an error about the subscription, click in `OK`:

![proxmox_12](static/proxmox/proxmox_12.png)

Now we're going fix the error we saw earlier and update the Proxmox node.

Choose `proxmox` in the side menu and then click in `Repositories`. Disable the ceph and pve-enterprise repositories:

![proxmox_13](static/proxmox/proxmox_13.png)

Click in add and choose the `No Subscription` option:

![proxmox_14](static/proxmox/proxmox_14.png)

Now to update the Proxmox node click in `Updates` and `Refresh`. And finally click in `Upgrade`:

![proxmox_15](static/proxmox/proxmox_15.png)

I my case there's was a kernel update, so I had to reboot the machine clicking in `Reboot` in the top right corner:

![proxmox_16](static/proxmox/proxmox_16.png)

That's it, now we have a Proxmox server ready to use.

### PfSense

Now we can start to think about our network setup. For security we will add a firewall between the ISP Router and the Proxmox server. My firewall of choice is PfSense, but you can use any other firewall you want.

To segregate the traffic we will use VLANs. I like to configure the following:
 - Home:
    - ID: 10
    - DHCP: 10.0.10.1/24
    - Reason: Network for personal devices like computers, phones, etc.
 - Guest:
    - ID: 20
    - DHCP: 10.0.20.1/24
    - Reason: Network for guests, like friends and family.
 - IOT:
    - ID: 30
    - DHCP: 10.0.30.1/24
    - Reason: Devices that I don't trust like smart TVs, smart speakers, etc.
 - Management:
    - ID: 99
    - DHCP: 10.0.99.1/24
    - Reason: Network for servers and management interfaces.

Using PfSense inside Proxmox isn't the best solution considering that if the PfSense VM goes down we will lose the Proxmox access, but it's a risk I'm willing to take.

To install PfSense we need to download the ISO from the [official website](https://www.pfsense.org/download/). After it follow the instructions in the [link](https://docs.netgate.com/pfsense/en/latest/install/prepare-installer-media.html#decompress-the-installation-media) to decompress the media.

Click in `local(proxmox)`, `ISO Images` and finally in `Upload` and choose the ISO file you downloaded earlier:

![pfsense_1](static/pfsense/pfsense_1.png)

As we are here let's also configure the network interfaces. First we will create the trunk interface for our LAN. Click in `proxmox`, `System`, `Network`, `Create` and choose `Linux Bridge`. My bridge will be named `vmbr1`, the port will be `enp3s0` and the VLAN aware option will be checked. Click in `Create`:

![pfsense_2](static/pfsense/pfsense_2.png)

Now we will create the interfaces for the VLANs. Click in `Create` and choose `Linux VLAN`. The name will be `vmbr1.10` and the description `HOME_VLAN`. Make sure `Autostart` is checked and click in `Create`:

![pfsense_3](static/pfsense/pfsense_3.png)

I will do the same for the other VLANs. The end result should be like this:

![pfsense_4](static/pfsense/pfsense_4.png)

Click in `Apply Configuration` to apply the changes.

Now let's create the VM. Click in `Create VM`. The configuration will be:
 - Name: pfsense
 - VM ID: 100
 - ISO Image: pfSense-CE-2.7.0-RELEASE-amd64.iso
 - Disk size: 16GB
 - CPU Cores: 2
 - Memory: 2048MB

Click in finish to create the VM.