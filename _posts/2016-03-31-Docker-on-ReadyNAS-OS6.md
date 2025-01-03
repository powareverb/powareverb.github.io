---
layout: post
title: Running Docker on ReadyNAS OS6 VM - From start to finish
published: true
---

As it turns out, not as difficult as I thought.  On a VM at least...

[Vote for support of Docker on ReadyNAS on the Ideas Exchange](https://community.netgear.com/t5/Idea-Exchange-for-ReadyNAS/Support-Docker-on-ReadyNAS-OS-6/idi-p/1069851#M234)

## Running Docker on ReadyNAS OS6 - From start to finish

So, not content with running Docker on All-The-Pis, I thought that I'd have a bash at running Docker on my new ReadyNAS (x86).  This thing is a beast, and it'd be awesome to run some long term containers on it.  
In the short term, I've instead started testing the idea by using the ReadyNAS VHD provided by NetGear.  Awesome to check things out without breaking/bricking the existing NAS.

### Why Docker?

I think Docker is a perfect fit for running slightly more custom apps on the NAS - heck, you could even run the ReadyNAS plugin system as Docker images if you liked.

- It gives a solid, simple way of compartmentalising applications away from a host - e.g. the ReadyNAS itself.
- It lets you run these things fast, pretty much bare metal fast in comparison to a VM.
- It's already well used and has a large library of images, support, etc.

I'm thinking of using it for adding things like

- MQTT messaging server
- SyncThing
- Distributed file storage system
- Distributed database (e.g. Crate.io)
- A Docker container repository (for x86 and my Arm images)

### Getting a VM started

So on a dev rig with VirtualBox installed, I download the VM and get started.
Browse to: <http://apps.readynas.com/pages/?page_id=143> and grab the VM file.

```sh
wget http://apps.readynas.com/download/ReadyNASOS-6.4.2-x86_64.vmdk
```

Follow the instructions (on the ReadyNAS page) to get a running VM on VirtualBox before going any further.  In my case, I had to reboot to turn off HyperV, as HyperV and VirtualBox don't play well when running 64 bit VMs.

### Upgrade to latest beta

Once you've got the VM running, I'd take a snapshot and then upgrade to the latest beta.  You can grab the upgrade to 6.5.0 Beta 2 here:
<https://community.netgear.com/t5/ReadyNAS-Beta-Release/ReadyNASOS-6-5-0-T338-Beta-2/m-p/1059936#U1059936>

Download the relevant img file, go to your virtual NAS in a browser and upgrade to 6.5.0.

- Browse to e.g. <https://192.168.1.100/admin/>
- Manual install firmware
- Select the image file you just downloaded

This will take a while, so sit back and enjoy the blinkenlights.  Grab a coffee or something.  Once that's done, make sure you have SSH enabled on the Virtual NAS (on the Admin -> Settings screen) and SSH into the VM.  

```sh
uname -a
# You should get something matching the new kernel, e.g.:
linux-4.1.19-x86_64
```

If so, then step 4 - profit!

### Getting setup with the latest Kernel

So Docker requires a few kernel options unfortunately missing from the standard build NetGear has given us.  The latest beta has many options selected that were previously missing (NAT and networking related), but there's still a few that Docker needs to run but are missing.  And unfortunately, some of those options can't be compiled as modules, meaning you have to do a full kernel rebuild.

The options in particular we need are in this gist:
<https://gist.github.com/powareverb/ca3de1df3ca83cebde23c5807edb8325>

There are likely some other configs which would be useful, but this will get us started.

Lets start by grabbing the kernel and getting set up.  SSH into the Virtual NAS and continue with the following.

```sh
mkdir /data/Documents/kernel-rebuild
cd /data/Documents/kernel-rebuild
wget https://www.readynas.com/download/GPL/readynasos/6.5.0/linux-4.1.19-178-x86_64.tar.xz
tar -xf linux-4.1.19-178-x86_64.tar.xz

# If you have an existing ln to linux kernel, remove first
# rm /usr/src/linux
ln -s /data/Documents/kernel-rebuild/linux-4.1.19-x86_64 /usr/src/linux

cd /usr/src/linux
make mrproper
cp arch/x86/configs/defconfig.x86 .config
```

#### Configuring the Docker compatible kernel

You should now have a good to go kernel tree. We need to merge in the diff I mentioned earlier:

```sh
# Haven't checked this for syntax yet, but this is the general idea
wget https://gist.githubusercontent.com/powareverb/ca3de1df3ca83cebde23c5807edb8325/raw/d3b9621cd3863425db0e0993445dbe575bbf4d43/.config-readynas.diff
patch .config .config-readynas.diff
cat .config | grep CONFIG_POSIX_MQUEUE
# You should get something like:
# CONFIG_POSIX_MQUEUE=y
```

If you want to check other kernel options for Docker, turns out there's a script for that.

```sh
#Docker missing kernel modules check:
wget https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh
chmod +x ./check-config.sh
./check-config.sh
```

This will list any kernel options that are needed but missing from the default kernel config location (e.g. /usr/src/linux/.config).  Will also tell you many of the optional kernel flags you can add too.

#### Compiling the Docker compatible kernel

This one is easy... Ish :)

```sh
# Install some build tools
apt-get install build-essential libncurses5-dev bc

# Make our kernel
make prepare
make modules_prepare
make 
make modules_install
depmod -a 
```

#### Installing the new kernel

> *WARNING: These instructions are for installing the kernel on a VM, installing a new kernel on a hardware NAS is problematic!*

```sh
#Still in kernel dir?
cd /usr/src/linux
ls -la arch/x86/boot/bzImage
mount /dev/sda1 /mnt/tmp
ls -la /mnt/tmp
cp /mnt/tmp/KERNEL /mnt/tmp/KERNEL-OLD
cp arch/x86/boot/bzImage /mnt/tmp/KERNEL
```

After that's done, you should be able to reboot into your new kernel!  To check:

```sh
modprobe configs
lsmod

# The configs module should be listed
zcat /proc/config.gz
```

### A few other initial prerequsites

We need a copy of a newer version of init-system-helpers, as the version on the ReadyNAS OS is too old.  To achieve this, I've used the version from wheezy-backports, by doing the following:

```sh
nano -w /etc/apt/sources.list

#Add the following line
deb http://mirrors.kernel.org/debian wheezy-backports main

#Then run the following
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
apt-get update
apt-get install perl
apt-get install init-system-helpers/wheezy-backports
```

### Installing Docker itself

```sh
apt-get install perl git wget
apt-get install aufs-tools cgroupfs-mount cgroup-bin
apt-get install docker-engine
```

### Testing it all out

Easiest thing here is to check:

```sh
systemctl status docker-engine
# Should be running

# Test docker out
docker version

# Should return e.g.
Client:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 15:48:06 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 15:48:06 2016
 OS/Arch:      linux/amd64
```

With that confirmed, might as well run a docker image!

```sh
root@nas-virtualbox:~# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
03f4658f8b78: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
Status: Downloaded newer image for hello-world:latest

Hello from Docker.
This message shows that your installation appears to be working correctly.
```

### Fin

As usual, feel free to contact me if you have any comments or questions!

[Vote for support of Docker on ReadyNAS on the Ideas Exchange](https://community.netgear.com/t5/Idea-Exchange-for-ReadyNAS/Support-Docker-on-ReadyNAS-OS-6/idi-p/1069851#M234)
