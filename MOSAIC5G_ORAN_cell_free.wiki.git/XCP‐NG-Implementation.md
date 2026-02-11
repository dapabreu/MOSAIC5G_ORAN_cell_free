# Intro

XCP-ng is an open source type 2 hypervisor with capabilities to create and manage Virtual Machines, from a remote host. XO (XenOrchestra) is the web-based management platform, virtual application or "dashboard" of XCP-ng in which we can perform various operations on VM's, storage units, network, create templates, etc.

In the context of 5G and Mosaic5G/OpenAirInterface, this tool can be a valuable asset to remotely insert docker files and test them, without using personal resources.

# Implementation

## Tutorial followed

[Install XCP-ng | XCP-ng documentation](https://docs.xcp-ng.org/installation/install-xcp-ng/)

## Enabling SSH for remote communications

[Enable SSH on XCP-ng](https://support.citrix.com/s/article/CTX238295-how-to-enabledisable-ssh-on-xenserver-host?language=en_US)

## Web UI Installation (XOA)

*Note: the connection to the DEI VPN already needs to be made - OpenVPN was used*

[Xen Orchestra Web UI | XCP-ng Documentation](https://docs.xcp-ng.org/management/manage-at-scale/xo-web-ui/)

A virtual application that should appear in the form of a new VM in the dashboard. Done through this link: https://vates.tech/deploy/ (as root).

From now on, we can access the dashboard via the link: https://[ip_da_máquina].

## Installation of XCP-ng Center (optional)

https://github.com/xcp-ng/xenadmin

> *This software is deprecated*

## Download ISO ubuntu 22.04 LTS

[Ubuntu 22.04.5 LTS (Jammy Jellyfish)](https://releases.ubuntu.com/jammy/)

## Insert a new .iso file remotely

**First Step:** Create a new storage unit

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/xcp-ng img/Screenshot_3.png" alt="flexric" width="500">


**Second Step:** Configure storage

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/xcp-ng img/Screenshot_2.png" alt="flexric" width="500">


**Third Step:** Import a new disk (this will be our . iso)

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/xcp-ng img/Screenshot_4.png" alt="flexric" width="500">


**Fourth Step:** Drag and drop the .iso file

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/xcp-ng img/Screenshot_5.png" alt="flexric" width="500">


# Additional Steps:

## Enabling SSH on the VM (ubuntu)

```bash
sudo apt update
sudo apt upgrade
sudo apt install openssh-server
systemctl enable ssh
systemctl start ssh
vi /etc/ssh/sshd_config
  #Change the PermitRootLogin to ON
systemctl restart ssh
  # Always login with User and then Root.
```

## Install Docker and Docker-Engine


```bash
sudo apt install -y git net-tools putty

# install docker for ubuntu.
# https://docs.docker.com/engine/install/ubuntu/
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt install docker-compose

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose docker-compose-plugin

# Add your username to the docker group, otherwise you will have to run in sudo mode.
sudo usermod -a -G docker $(whoami)

reboot 
```

# Useful Links

[Virtual Machines (VMs) | XCP-ng Documentation](https://docs.xcp-ng.org/vms/)

[xe command reference | XCP-ng Documentation](https://docs.xcp-ng.org/appendix/cli_reference/)

# Problems

## Compatibility issues w/ processor architecture

- After starting the eNodeB and UE - both containers are exited and the gNB logs report this:

```
== Starting gNB soft modem
Additional option(s): --sa -E --rfsim --log_config.global_log_options level,nocolor,time
/opt/oai-gnb/bin/nr-softmodem -O /opt/oai-gnb/etc/gnb.conf --sa -E --rfsim --log_config.global_log_options level,nocolor,time
[INFO tini (1)] Spawned child process '/opt/oai-gnb/bin/entrypoint.sh' with pid '6'
[INFO tini (1)] Main child exited with signal (with 'Illegal instruction' signal)
```

Cause: Current OpenAirInterface.org software requires Intel architecture-based PCs
for eNB or UE targets. This requirement is due to optimized DSP functions
which make heavy use of integer SIMD instructions (SSE, SSE2, SSS3, SSE4, and AVX2).
The software has currently been tested on the following processor families:

Generation 3/4/5/6 Intel Core i5,i7
Generation 2/3/4 Intel Xeon
Intel Atom Rangeley, E38xx, x5-z8300

Our server's CPU is from a previous generation, we can check it using the command (executed directly on the hypervisor) ```x and host-cpu-info```.

- [CPU Specs Info](https://ark.intel.com/content/www/us/en/ark/products/47925/intel-xeon-processor-e5620-12m-cache-2-40-ghz-5-86-gt-s-intel-qpi.html)

- [OAI System Requirements](https://gitlab.eurecom.fr/oai/openairinterface5g/-/wikis/OpenAirSystemRequirements)


***

## Logs are filling up the space in disk

- /var/ folder got filled with 50gb of files - maybe implement logrotate mechanism, and evaluate the necessary space for the docker-compose and the implications of running docker-compose various times. 

[View history of commands run in terminal - Ask Ubuntu](https://askubuntu.com/questions/624848/view-history-of-commands-run-in-terminal)

[logrotate(8) - Linux Manual Page (man7.org)](https://man7.org/linux/man-pages/man8/logrotate.8.html)

[Rotate and archive logs with the logrotate command Linux | Opensource.com](https://opensource.com/article/21/10/linux-logrotate)

[How To Manage Logfiles with Logrotate on Ubuntu 20.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-manage-logfiles-with-logrotate-on-ubuntu-20-04)


> "Solved" using one of the snapshots created after configuring the OAI docker-compose and restoring the space.

> Potentially, this problem may be caused by leaving the containers running

***


## Lack of space in the VMs Disk

Firstly, always run commands to remove the **unused** volumes that build up over time, images and networks using `docker network prune`, `docker volume prune` or `docker image prune`.

A solution is to add a new disk, using the XenOrchestra virtual appliances. After creating the new disk (make sure you are using the correct *pool*) we can connect it to the VM and `sudo reboot` if the machine is on.

- Run: `sudo ls /dev/xvd`
  - With this we can see the new added disk and all of the available partitions. 
- Additionally: `sudo lvscan`
  - Check all of the logical volumes available in the system. You should only have one, that corresponds to the current used disk, or your main mount point.

The main objective now is to add the new disk, that should correspond to **xvdb**, to the just now mentioned logical volume group **ubuntu-lv**. We will then expand that new logical volume to use the full available space of the 2 disks combined.

```bash
# use your secondary disk, in our case it's xvdb. This will initialize it as the disk of the physical volume
sudo pvcreate /dev/xvdb

>   Physical volume "/dev/xvdb" successfully created.

# Check if the disk was really added
sudo fdisk -l

# Add the second disk
sudo pvcreate /dev/xvdb

# let's now see all of the physical volumes, sizes and info
sudo pvs

>   PV         VG        Fmt  Attr PSize   PFree
  /dev/xvda3 ubuntu-vg lvm2 a--  <23.00g 11.50g
  /dev/xvdb            lvm2 ---   20.00g 20.00g

# Again, let's see all of the volume groups
sudo vgscan

>   Found volume group "ubuntu-vg" using metadata type lvm2

# Now, we will add xvdb (our secondary disk - the new) to the volume group, in order to have more space in that volume group
sudo vgextend ubuntu-vg /dev/xvdb

>  Volume group "ubuntu-vg" successfully extended

# list all available groups
sudo vgs

>   VG        #PV #LV #SN Attr   VSize  VFree
  ubuntu-vg   2   1   0 wz--n- 42.99g <31.50g

# we can also see the logical volumes and + info with: 
sudo lvscan

> ACTIVE            '/dev/ubuntu-vg/ubuntu-lv' [<11.50 GiB]

# Finally, let's expand the logical volume ubuntu-vg/ubuntu-lv using all of the free space available in the associated volume group
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

> Size of logical volume ubuntu-vg/ubuntu-lv changed from <11.50 GiB (2943 extents) to 42.99 GiB (11006 extents).

# resizing the filesystem
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

# we can see the augmented space
df -h

> /dev/mapper/ubuntu--vg-ubuntu--lv   43G   11G   30G  27% /
```
