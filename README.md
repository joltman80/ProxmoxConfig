# ProxmoxConfig

Configuration of my Proxmox Server

## Install

### Prepping Install Media

Download the latest Proxmox ISO from this [link](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso).  You can either use IPMI to mount the ISO to the host over the network, or write the ISO to a USB stick of sufficient size.  Following the [official instructions](https://pve.proxmox.com/wiki/Prepare_Installation_Media), follow these commands:

Plug in the USB stick and find the correct /dev/sdX path:

```console
lsblk
```

You might need to delete all the partitions on the USB stick first.  Use Gnome Disks for that.  Once the partitions have been deleted, run the following command to write the ISO to the USB stick:

```console
dd bs=1M conv=fdatasync if=./proxmox-ve_*.iso of=/dev/XYZ
```

Once the USB stil is created, eject it and plug it into the server.

### Installing on Host

Connect to the host's IPMI web page and start up the iKVM CONSOLE.  Boot the machine and enter into BIOS (Typically DEL or F2).  Verify the boot order for the host so that the main disk(s) will be booted first.  When you go to save the BIOS config, choose the SAVE option that also includes a next boot override to boot of the USB stick.

When the server boots again, it will use the USB stick to boot the Proxmox Installer.  Proceed through the GUI installer.  Make sure to choose ZFS MIRROR on your designated OS disks.  Once the installer is finished, reboot the node.  Take out the USB stick before the BIOS screen.

## Initial Host Configuration

While still using the IPMI iKVM Console login to the Proxmox CLI using the root creds you created during the install.

## Host Configuration

Follow the steps below to configure the host in preparation for installing containers and VMs.

### Basic Network Configuration

The interfaces on the host have to be configured into bonds and/or bridges for use by containers and VMs.  The most redundant network configuration would be bond of two or more NICs together in an 802.3ad LACP group.  Your switch has to support this configuration.  Typically, the Linux host (Proxmox) will be the "active" side of the connection.  A Cisco switch can be configured as an ACTIVE side (mode on) or PASSIVE side (mode passive).  If the switch and host are both set ACTIVE, the portchannel interface and bond will come up.  These are the commands to run on the Cisco switch to create an LACP etherchannel.

First you must configure the HASH type that the LACP etherchannel will use:

```console
conf t
port-channel load-balance src-dst-ip
end
```

```console
conf t
int gX/X/X
description Proxmox Host en01
switchport mode trunk
switchport trunk encapsulation dot1q
switchport trunk allowed vlan x, y-z
switchport nonegotiate
channel-group NUMBER mode passive
```

Run that command on the two or more interfaces you will connect to the Proxmox host.  Now configure the actual Port Channel NUMBER interface.

```console
interface PortChannel-NUMBER
description Proxmost Host LACP
 switchport mode trunk
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10-13,19-24,666,997,999
 switchport nonegotiate
```

Don't forget to write your changes to the switch's memory.

```console
wri
```

Now it is time to configure the network settings on the Proxmox host.  Look at this repo's configuration file under /etc/network/interfaces.  This will have the config you need to enable LACP on the Proxmox host.  Once you've copied the config into the host and restart the network stack, from the host, try pinging your local gateway.  Then try pinging an external IP.

### SSH Access

Now that the PVE host is accessible over the network, you will need to configure secure SSH access.  The best method for this is to create SSH keys your laptop and copy them to the PVE root account.

```console
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@pve01.theoltmanfamily.net
```

You can then disable Password based SSH access for the root user.

### Optimize Host Using Post-Install Script

> [!NOTE]
> ALWAYS examine a script before blindly running it.  It could contain malicious code and a few moments reviewing the script will help you learn what it's actually doing.

There are community developed scripts that will assist you with installing CTs or VMs on your new host.  These scripts are hosted [here](https://community-scripts.github.io/ProxmoxVE/scripts).  This is the [link](https://github.com/community-scripts/ProxmoxVE) for the base repo for these scripts.  Specifically, we're going to use the [Post-Install](https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/post-pve-install.sh) script.  This script will update the APT SOURCE list and remove any references to the Enterprise repos, correct the Debian repos, etc.  It will also remove the Subscription nag screen on PVE login.  It will then perform an apt update and apt dist-upgrade and finally prompt to reboot the host.  You can get and run the script by SSHing to the host and running the following command on the host:

```console
bash -c "$(wget -qLO - https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/post-pve-install.sh)"
```

Choose yes for most options, except the TESTING repo and anything else that you don't want.

### Software Defined Network Configuration

When you create a CT or VM, you will want to assign a network to it.  You can do that by assigning a VMBR and then adding a VLAN tag.  This can get confusing if you don't remember the VLAN IDs.  To get around that, Proxmox can use SDN to configure named Zones that have named VNETs that are already assigned the VLAN ID.  All you have to do is name the VNET appropriately.  First, create the ZONE then create the VNET for that Zone.  You can do this from the GUI using these commands.

```console
pvesh create /cluster/sdn/zones/ --type vlan --zone HomeLAN --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet HomeLAN --alias "Home Client Wired VLAN" --zone HomeLAN --tag 10

pvesh create /cluster/sdn/zones/ --type vlan --zone HomeVoIP --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet HomeVoIP --alias "Home VoIP Phones VLAN" --zone HomeVoIP --tag 11

pvesh create /cluster/sdn/zones/ --type vlan --zone HomeWLAN --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet HomeWLAN --alias "Home Client Wireless VLAN" --zone HomeWLAN --tag 12

pvesh create /cluster/sdn/zones/ --type vlan --zone VidConf --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet VidConf --alias "Home Video Conferencing Devices" --zone VidConf --tag 13

pvesh create /cluster/sdn/zones/ --type vlan --zone Media --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet Media --alias "Home Media VLAN" --zone Media --tag 19

pvesh create /cluster/sdn/zones/ --type vlan --zone Server --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet Server --alias "Home Server VLAN" --zone Server --tag 20

pvesh create /cluster/sdn/zones/ --type vlan --zone IoT --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet HomeIoT --alias "Home Automation VLAN" --zone IoT --tag 21

pvesh create /cluster/sdn/zones/ --type vlan --zone DMZ1 --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet DMZ1 --alias "Home DMZ 1" --zone DMZ1 --tag 22

pvesh create /cluster/sdn/zones/ --type vlan --zone DMZ2 --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet DMZ2 --alias "Home DMZ 2" --zone DMZ2 --tag 23

pvesh create /cluster/sdn/zones/ --type vlan --zone MGMT --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet MGMT --alias "Firewall/Switch Management" --zone MGMT --tag 24

pvesh create /cluster/sdn/zones/ --type vlan --zone Guest --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet Guest --alias "Home Guest VLAN" --zone Guest --tag 666

pvesh create /cluster/sdn/zones/ --type vlan --zone DMZSIP --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet DMZSIP --alias "SIP Gateway DMZ" --zone DMZSIP --tag 997

pvesh create /cluster/sdn/zones/ --type vlan --zone PubInet --bridge vmbr0 --ipam pve
pvesh create /cluster/sdn/vnets --vnet PubInet --alias "Public Internet VLAN" --zone PubInet --tag 999
```

Deleting a VNET:

```console
pvesh delete /cluster/sdn/vents/VNETNAMEHERE
```

Deleting a Zone:

```console
pvesh delete /cluster/sdn/zones/ZONENAMEHERE
```

You can then save these SDN changes with this command:

```console
pvesh set /cluster/sdn
```

### Storage Configuration

Proxmox will need storage created and defined.  To do that, you must create local storage on the host.  I prefer ZFS file systems.

> [!NOTE]
> If you have drives that already had a partition, you won't see these drives as available when you create new storage.  Delete the partitions on these drives first by navigating to the host name > Disks.  Hightlight the disk and click on WIPE.

You can create a storage pool from the web GUI by navigating to HOST > DISKS > ZFS.  Click on CREATE: ZFS.  Give the following parameters:

NAME
RAID LEVEL (recommend MIRROR for 2 disks, or RAIDZ2 for 4+ disks)
COMPRESSION: ON
ASHIFT: 12 (DEFAULT)

In the Web GUI click on DATACENTER > STORAGE.  Look at the new storage device you just created.  Verify that it is setup for DISK IMAGE, CONTAINER.

You can also create a new DIRECTORY that can contain ISOs and CT Templates and store that on the ZFS pool you created.  From the STORAGE menu, click on ADD > DIRECTORY.  Fill out the following fields:

ID:  Make this a name with no spaces, ex:  ISO_CT
DIRECTORY: /mnt/ZFSPOOLNAMEHERE
CONTENT:  Click the drop down and choose VZDump, ISO Image, Container Templates

Click on ADD.

> [!NOTE]
> You can also use the storage on the Proxmox OS root pool for ISOs and CT Templates.  That is not recommended as it will be deleted when you upgrade the OS to a new PVE version.

### Special Device Drivers

Sometimes it will be required to install special drivers on the Proxmox host in order to share that device with a container.  For this host, we'll be configuring the mini PCIe Coral Dual TPU adapter for use in a Frigate unpriviledged LXC.  This adapter is installed on a PCIe x1 card that is specifically designed for the dual Coral TPU.  We will be following the instructions listed [here](https://github.com/blakeblackshear/frigate/discussions/5448#discussioncomment-8855247) and [here](https://github.com/google-coral/edgetpu/issues/808#issuecomment-2067170584).  [This](https://forum.proxmox.com/threads/update-error-with-coral-tpu-drivers.136888/page-5) is also a good thread on the PCIe Coral TPU.  We will be substituting in a different [GitHub repo](https://github.com/feranick/gasket-driver) for the Gasket Driver.

#### Build the gasket-dkms on Proxmox

We need to download software on the PVE host in order to compile the DKMS module for the Coral TPU.  Follow these steps:

```console
cd ~/
apt update
apt upgrade
apt install pve-headers-$(uname -r) proxmox-default-headers dh-dkms devscripts git dkms lsb-release sudo
mkdir Packages
cd Packages
git clone  https://github.com/feranick/gasket-driver.git
cd gasket-driver
debuild -us -uc -tc -b -d
cd ..
dpkg -i gasket-dkms_1.0-18.2_all.deb
reboot
```

After the reboot, check to see if the gasket module was installed by verifying if you have two APEX devices:

```console
ls /dev/apex*
```

If you see both devices like the below, you've sucessfully installed the module!

```console
root@pve01:~# ls /dev/apex*
/dev/apex_0  /dev/apex_1
root@pve01:~#
```

## Build A Management Container

We will be building an unpriviledged container for use in managing the host.  This container will be built using Debian Bookworm as the template and then installing docker and docker compose.  We will then install Portainer as the management tool for the rest of the LXCs that will be built.  The Portainer Agent will then be deployed on the other LXCs on the node so they can be managed by the main Portainer container.

### Download the Container Template

From the Web GUI, click on the PVE host name, scroll down to the Storage Director you created called ISO_CT.  Select CT Templates from the menu and click on the TEMPLATES button.  Choose the Debian 12 Bookworm template and download it.

### Create the Template

From the top right hand corner of the WebGUI click on CREATE CT.  Fill out all the fields as needed.  For the initial management template, choose 2 CPUs, 2GB of RAM, 4GB of disk space and choose UNPRIVILEDGED.  Create a new ROOT password and copy in your SSH public key.

### Install Docker and Compose

SSH into the container and follow the [Docker guide](https://docs.docker.com/engine/install/debian/) for installing Docker in Debian Bookworm.

```console
# Add Docker's official GPG key:
apt-get update
apt-get dist-upgrade
reboot
apt-get install ca-certificates curl sudo
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Now create a folder for all docker container configs and the associated compose.yaml files.

```console
mkdir /docker
```

You can now use VS Code to connect to this container and create your compose files.
