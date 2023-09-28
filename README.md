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

![proxmox_1](static/proxmox/1.png)

In my case the ISP Router is using 192.168.1.1/24 for the DHCP server. I will connect the interface enp2s0 to the router and use the IP 192.168.1.100 for the Proxmox server. We will change this IP later to be served by the internal OPNsense server.

The first thing necessary is to configure the BIOS of the machine to enable virtualization. In my case, I had to enable the following options:
 - Intel Virtualization Technology
 - Intel VT-d
 - Intel VT-x

After that, we can proceed with the installation of Proxmox. To do this, we need to download the ISO from the [official website](https://www.proxmox.com/en/downloads/category/iso-images-pve) and burn it to a USB stick. Then we need to boot from the USB stick and follow the installation steps.

This is the screen when you first boot from the USB stick. Choose `Install Proxmox VE (Graphical)`:

![proxmox_2](static/proxmox/2.jpg)

Click to accept the license agreement:

![proxmox_3](static/proxmox/3.jpg)

Choose where to install the server, in my case I will choose the entire nvme disk with the ext4 file system:

![proxmox_4](static/proxmox/4.jpg)

Choose the country, time zone and keyboard layout:

![proxmox_5](static/proxmox/5.jpg)

Choose the password for the root user:

![proxmox_6](static/proxmox/6.jpg)

Choose the NIC interface, hostname, IP address, gateway and DNS server. In my case this will be:
 - Interface: enp2s0
 - Hostname: proxmox.mfurlanetto.com
 - IP: 192.168.1.100/24
 - Gateway: 192.168.1.1
 - DNS: 1.1.1.1

![proxmox_7](static/proxmox/7.jpg)

Click in `Install` to start the installation.

![proxmox_8](static/proxmox/8.jpg)

The installation will take a few minutes:

![proxmox_2](static/proxmox/9.jpg)

When the installation is finished, remove the USB stick and reboot the machine.

When the machine boots, you will see the following screen with the URL to access the server:

![proxmox_10](static/proxmox/10.jpg)

After acessing the URL you will probably see a warning about the self signed certificate, click to proceed anyway:

![proxmox_11](static/proxmox/11.png)

Login with the root user and the password you defined during the installation:

![proxmox_12](static/proxmox/12.png)

After login you will see an error about the subscription, click in `OK`:

![proxmox_13](static/proxmox/13.png)

Now we're going fix the error we saw earlier and update the Proxmox node.

Choose `proxmox` in the side menu and then click in `Repositories`. Disable the ceph and pve-enterprise repositories:

![proxmox_14](static/proxmox/14.png)

Click in add and choose the `No Subscription` option:

![proxmox_15](static/proxmox/15.png)

Now to update the Proxmox node click in `Updates` and `Refresh`. And finally click in `Upgrade`:

![proxmox_16](static/proxmox/16.png)

I my case there's was a kernel update, so I had to reboot the machine clicking in `Reboot` in the top right corner:

![proxmox_17](static/proxmox/17.png)

That's it, now we have a Proxmox server ready to use.

### OPNsense

Now we can start to think about our network setup. For security purposes we will add a firewall between the ISP Router and the Proxmox server. My firewall of choice is OPNsense, but you can use any other firewall you want.

Like I said earlier my Mini PC has two NICs, `enp2s0` and `enp3s0`. The first one will be our WAN and the second one our LAN. When we installed Proxmox created by default a bridge called `vmbr0` that is using the `enp2s0` interface. We will created another one for `enp3s0` called `vmbr1` that will be our trunk port.

To segregate the traffic we will use VLANs. I like to configure the following:
 - HOME_VLAN:
    - ID: 10
    - DHCP: 10.0.10.1/24
    - Reason: Network for personal devices like computers, phones, etc.
 - GUEST_VLAN:
    - ID: 20
    - DHCP: 10.0.20.1/24
    - Reason: Network for guests, like friends and family.
 - IOT_VLAN:
    - ID: 30
    - DHCP: 10.0.30.1/24
    - Reason: Devices that I don't trust like smart TVs, smart speakers, etc.
 - MGMT_VLAN:
    - ID: 99
    - DHCP: 10.0.99.1/24
    - Reason: Network for servers and management interfaces.

Using OPNsense inside Proxmox isn't the best solution considering that if the OPNsense VM goes down we will lose the Proxmox access, but it's a risk I'm willing to take.

To install OPNsense we need to download the ISO from the [official website](https://opnsense.org/download/).

Click in `local(proxmox)`, `ISO Images` and finally in `Upload` and choose the ISO file you downloaded earlier:

![opnsense_1](static/opnsense/1.png)

As we are here let's also configure the network interfaces. First we will create the trunk interface for our LAN. In the left panel click in `proxmox`, `System`, `Network`, `Create` and choose `Linux Bridge`. My bridge will be named `vmbr1`, the port will be `enp3s0` and the VLAN aware option will be checked. Click in `Create`:

![opnsense_2](static/opnsense/2.png)

Now we will create the interfaces for the VLANs. Click in `Create` and choose `Linux VLAN`. The name will be `vmbr1.10` and the description `HOME_VLAN`. Make sure `Autostart` is checked and click in `Create`:

![opnsense_3](static/opnsense/3.png)

I will do the same for the other VLANs. The end result should be like this:

![opnsense_4](static/opnsense/4.png)

Click in `Apply Configuration` to apply the changes.

Now let's create the VM. In the top bar click in `Create VM`. 

In General use `opnsense` for name:

![opnsense_5](static/opnsense/5.png)

In OS choose the OPNsense ISO we uploaded earlier:

![opnsense_6](static/opnsense/6.png)

In System don't change anything:

![opnsense_7](static/opnsense/7.png)

In Disks change size to 16GB:

![opnsense_8](static/opnsense/8.png)

In CPU change cores to 2:

![opnsense_9](static/opnsense/9.png)

In Memory change size to 2048MiB:

![opnsense_10](static/opnsense/10.png)

In Network we can only choose one nic, so leave `vmbr0` for now. Make sure to disable firewall:

![opnsense_11](static/opnsense/11.png)

Make sure the `Start after created` option is not checked and click in `Finish`.

![opnsense_12](static/opnsense/12.png)

In the left bar click in `opnsense` and then in `Hardware`. Click in `Add` and choose `Network Device`:

![opnsense_13](static/opnsense/13.png)

Choose `vmbr1`, disable firewall and click in `Add`:

![opnsense_14](static/opnsense/14.png)

So `vmbr0` will be used for the WAN interface and `vmbr1` for the LAN interface. Now we can start the VM. Click in `Start` in the top bar and enter the console:

![opnsense_15](static/opnsense/15.png)

Wait for the OS to boot and login with:
 - Username: installer
 - Password: opnsense

![opnsense_16](static/opnsense/16.png)

Choose the keyboard layout, in my case `Brazilian (accent keys)`:

![opnsense_17](static/opnsense/17.png)

Choose `Install (UFS)`:

![opnsense_18](static/opnsense/18.png)

Select `da0` for disk:

![opnsense_19](static/opnsense/19.png)

Select `YES`: to continue:

![opnsense_20](static/opnsense/20.png)

Wait for the installation to finish. Choose `Change root password`:

![opnsense_21](static/opnsense/21.png)

Choose a strong password. Now select `Complete Install` to reboot:

![opnsense_22](static/opnsense/22.png)

Don't forget to remove the installation media:

![opnsense_23](static/opnsense/23.png)

Wait for the VM to restart. After the restart you will see the following screen:

![opnsense_24](static/opnsense/24.png)

Login using:
 - Username: root
 - Password: the password you defined earlier

![opnsense_25](static/opnsense/25.png)

OPNsense choosed the wrong interfaces for WAN and LAN and I want to change the IP address for LAN. Let's fix that.

Choose option `2) Set interface IP address`:

![opnsense_26](static/opnsense/26.png)

Now OPNsense will ask some questions:

 - Enter the number of the interface to configure: 1
 - Configure IPv4 address LAN interface via DHCP? (y/N): N
 - Enter the new LAN IPv4 address: 10.0.0.1
 - Enter the new LAN IPv4 subnet bit count (1..32): 24
 - For a LAN, press <ENTER> for none: Enter
 - Configure IPv6 address LAN interface via WAN tracking? (Y/n): n
 - Configure IPv6 address LAN interface via DHCP6? (y/N): N
 - Enter the new LAN IPv6 address: Enter
 - Do you want to enable the DHCP server on LAN? (y/N): y
 - Enter the start address of the IPv4 client address range: 10.0.0.50
 - Enter the start address of the IPv4 client address range: 10.0.0.254
 - Do you want to change the web GUI protocol from HTTPS to HTTP? (y/N): N
 - Do you want to generate a new self-signed web GUI certificate? (y/N): N
 - Restore web GUI accress defaults? (y/N): N

![opnsense_27](static/opnsense/27.png)