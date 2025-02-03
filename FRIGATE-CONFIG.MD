# Frigate Container Config

This document will explain the initial configuration of the Frigate LXC container.

## Prep the container

Build a container using the configuration steps from [README.md](./README.md).  Build the container with the following config:

>4 CPUs
>8192 MB RAM
>16GB OS Disk
>NET0 in Server VLAN
>NET1 in IoT VLAN (no gateway)
>UNPRIVILEDGED

### User Config

Boot the container.  Once the system is booted follow the steps from [README.md](./README.md) to update the OS and assign the correct timezone.  Create a new user called ```administrator```.

```console
useradd --uid 1000 --gid 1000 --user-group -groups users --create-home --home-dir /home/administrator --shell /bin/bash administrator
```

We didn't add the password for the new user to the above command for security reasons.  It will be visible to any user listing processes.  You should use the following command to create a password for the new user:

```console
echo 'administrator:mypass' | chpasswd
```

Shutdown the container.

### Gasket Module

Next, we have to passthrough the DKMS ```gasket``` module to the container.  Open an SSH session to the host and navigate to the PVE LXC config folder:

```console
cd /etc/pve/lxc
```

Based on the number next to the container name in the WebGUI, you will want to edit that file in the directory above:

```console
nano 101.conf
```

At the end of the file, insert the following line and save & exit the file:

```console
linux.kernel_modules: gasket
```

### Hardware Sharing (Passthrough)

Now, we need to share (passthrough) the Coral TPU device and Intel GPU to the container.  Click on the CONTAINER in the PVE WebGUI, then select RESOURCES in the middle menu.  Click the ADD button and choose DEVICE PASSTHROUGH.  Enter in ```/dev/apex_0```.  Create another DEVICE PASSTHROUGH and add ```/dev/apex_1```.  Find the correct device names for ```/dev/dri/card#``` and ```/dev/dri/rednerD12#``` using the following steps on the PVE HOST:

```console
lspci | grep VGA
```

Look at the PCI IDs of the Intel VGA card.  It will look similar to this:

```console
00:02.0 VGA compatible controller: Intel Corporation CoffeeLake-S GT2 [UHD Graphics 630]
09:00.0 VGA compatible controller: ASPEED Technology, Inc. ASPEED Graphics Family (rev 41)
```

Now, look in the /dev/dri/by-path folder and run an ```ls -lha``` to find the links to the path that has the PCI ID above and the ```card#``` and ```renderD12#``` devices.

```console
root@pve01:# cd /dev/dri/by-path
root@pve01:/dev/dri/by-path# ls -lha
total 0
drwxr-xr-x 2 root root 100 Feb  2 22:15 .
drwxr-xr-x 3 root root 120 Feb  2 22:15 ..
lrwxrwxrwx 1 root root   8 Feb  2 22:15 pci-0000:00:02.0-card -> ../card1
lrwxrwxrwx 1 root root  13 Feb  2 22:15 pci-0000:00:02.0-render -> ../renderD128
lrwxrwxrwx 1 root root   8 Feb  2 22:15 pci-0000:09:00.0-card -> ../card0
root@pve01:/dev/dri/by-path# 
```

Here we can see that the PCI path that has **02** in it has a card and render device of ```/card1``` and ```renderD128```.  That means that the devices you will be passing through to your containers are:

```console
/dev/dri/card1
/dev/dri/renderD128
```

Now we need to know what groups/gids are assigned to ```card1``` and ```renderD128```.

```console
root@pve01:/dev/dri# ls -lha
total 0
drwxr-xr-x  3 root root        120 Feb  2 22:15 .
drwxr-xr-x 18 root root       4.8K Feb  2 22:15 ..
drwxr-xr-x  2 root root        100 Feb  2 22:15 by-path
crw-rw----  1 root video  226,   0 Feb  2 22:15 card0
crw-rw----  1 root video  226,   1 Feb  2 22:15 card1
crw-rw----  1 root render 226, 128 Feb  2 22:15 renderD128
root@pve01:/dev/dri# 
```

Here we can see that ```root``` is the user owner and the group owners are ```video``` and ```render```.  We need to find their gids.

```console
root@pve01:~# cat /etc/group | grep video
video:x:44:
root@pve01:~# cat /etc/group | grep render
render:x:104:
root@pve01:~# 
```

Passthrough these devices to the Frigate container using the appropriate gids for each device.  Since we're running the same OS in the container as the host, the gids and names are the same.

```console
/dev/dri/card1,gid=44
/dev/dri/renderD128,gid=104
```

Boot the container and SSH in.  Verify that the ```gasket``` module has been loaded by running:

```console
root@frigate:~# lsmod | grep gasket
gasket                126976  1 apex
root@frigate:~# 
```

Verify that ```card1``` and ```renderD128``` have loaded and have their correct groups assigned:

```console
root@frigate:~# ls -lha /dev/dri/
total 0
drwxr-xr-x 2 root root         80 Feb  2 22:33 .
drwxr-xr-x 7 root root        540 Feb  2 22:33 ..
crw-rw---- 1 root video  226,   1 Feb  2 22:33 card1
crw-rw---- 1 root render 226, 128 Feb  2 22:33 renderD128
root@frigate:~# 
```

Using the instructions in the main README.md file, you must:
Configure the correct [timezone](./README.md/#basic-configuration)
[Install docker and docker compose](./README.md/#install-docker-and-compose)
