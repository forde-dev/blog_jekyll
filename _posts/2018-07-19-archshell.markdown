---
layout: post
title:  "How to install ArchLinux!"
date:   18-07-2018 14:37:40 +0200
img: archlinux.png
description: Tutorial on how to create an automated Archlinux installer shell script.
categories: ArchLinux, bash, Linux
comments: true
github: arch
gitdownload: arch/archive/master.tar.gz
---

Hello, this is my second blog post. In this blog i will be giving a tutorial on how to install ArchLinux.
What im going to do is create a bash shell script with all the nessasary commands needed to install
arch, then i will test and trouble shoot it using a virtual mashine until it works smoothly.
I will mostly follow the pecedure layed out by [Archlinux installation guide][guide] wehn creating this script.
I will do my best to make it as simple as possible, I also plan on making a youtube video explaining everything typed
here.

## 1. Step One.

#### Setting up,

I would recommend creating a directory(folder) or a [git repo][gitrepo] for this project.
If you are in linux and want to create a directory use the following command, just replace **"my_new_directory"** with what you want to call it.
{% highlight ruby %}
# example

mkdir my_new_directory
{% endhighlight %}

The file layout for this project is as follows, Just create these files.

> base.sh

> post.sh

NOTE: make sure that your text editor of choice is set to LF (unix line endings) and UTF-8
this will mostly not be a problem if your writing this on linux, however if your using Windows to
write this I'd recommend using [Atom][atom] or [Notepad ++][notpad].


#### First,

Open both **base.sh** and **post.sh** to initialise the script using a **Shebang** as follows,
i will also add *set -e* this option makes the shell script error out whenever a command errors out.
It's generally a good idea to have it enabled most of the time. but this is optional.

{% highlight ruby %}
#!/usr/bin/env bash

set -e
{% endhighlight %}

Just to give a bit more explaining on the **shebang** above which is the ***#!***, this tells the script where to find
the bash interpreter, typically people use ***#!/bin/bash*** but using ***#!/usr/bin/env bash*** lends you some
extra flexibility on different systems.


#### Next,
we need to guarantee that the system is in **UEFI** mode as the rest of this script depends on it.
If the archiso is ran in UEFI there will be a ***efi*** directory in the ***/sys/firmware***,
example the following path must be true for it to me **UEFI**,

> /sys/firmware/efi/

To test this in our script we can use the operator ***-d*** to test if it exists and use an **IF** statmemt to allow to program to run
or not. You can just copy the following piece of code and add it on to the **base.sh**,

{% highlight ruby %}
# this checks if the system is uefi
if [ ! -d "/sys/firmware/efi/" ]; then
	echo "This script only works in UEFI"
	exit 1
fi
{% endhighlight %}

## 2. Step Two.

#### Initialising variables,

Lets now continue editing the **base.sh** and create a few variables.
in case you don't know hwo to crate a variable in bash it works like the following,

{% highlight ruby %}
# example

VARIABLE="items"

# and to use the variable

mkdir ${VARIABLE}
{% endhighlight %}

So the variables we will need to add to the **base.sh** are as follows,

{% highlight ruby %}
# set a root password
ROOTPASSWORD="password"

# set a username for your user
USERNAME="user"

# set a password for your user
USERPASSWORD="userpassword"

# set a hostname for the system
HOSTNAME="computer"

# set the drive to install on (more on that in a moment)
DRIVE="/dev/sda"

# set keymap for the keyboard
KEYMAP="uk"

# set up what packages need to be installed (also more on this below)
PKG="base base-devel refind-efi wireless_tools nfs-utils ntfs-3g openssh pkgfile pacman-contrib mlocate mlocate alsa-utils bash-completion rsync"
{% endhighlight %}

In the above variables you can set them all to suit your own preference.
This in **IMPORTANT**, the **DRIVE** variable has being assigned to ***"/dev/sda"*** above,
you can change this if you want, but i recommend not, although the script that we are making will wipe all the contents
on the ***"/dev/sda"*** drive, i would also recommend disconnecting any other drives in the computer when doing this install,
no need to worrie about that just yet, the following will show you how you can see what drives are which.

{% highlight ruby %}
# example, to check drives in your system

lsblk
{% endhighlight %}

Heres an example of the output of the ***lsblk*** command.

![lsblk output][lsblk]

I will also give a closer look at the **PKG** variable, all the packages that assigned are optional, i have
picked these as they are what i wanted, you can check out your options [here][packages], if you dont understand this
dont worry too much, just use what i have provided above.

#### Finally,

We must set up a few last things before moving on to step three,
now lets sync the clocks to the systems local time using the following commands,

{% highlight ruby %}
# clocks
# this enables the time controller
echo "Setting local time"
timedatectl set-ntp true

# this syncs with the system clock
hwclock --systohc --utc
{% endhighlight %}

Also we must set up the keyboard to match the variable up above, its done as simple as this,

{% highlight ruby %}
# keyboard
# this loads the keyboard depending on your country as set in the variables
echo "Loading Uk Keymap for the keyboard"
loadkeys ${KEYMAP}
{% endhighlight %}

## 3. Step Three.

#### Partitioning,

We will create three Partitions out of the **DRIVE**,
+ efi
+ Swap
+ Root

I am going to use a tool called ***sgdisk***.
First we must wipe the drive and reformate it using the following,

{% highlight ruby %}
# this wipes and formates the drive
echo "# Wriping Drive and segergating"
sgdisk -Z ${DRIVE}
{% endhighlight %}

And now segergste the drive,

{% highlight ruby %}
# this optimumises the partition
sgdisk -a 2048 -o ${DRIVE}

# this makes the EFI partition
echo "Setup UEFI Boot Partition"
sgdisk -n 1:0:+512M -t 1:ef00 -c 1:"EFI System Partition" ${DRIVE}

# this makes the EFI partition a vfat filesystem
mkfs.vfat ${DRIVE}1

# this create the swap partition
echo "Setup Swap"
sgdisk -n 2:0:+2G -t 2:8200 -c 2:"Swap Partition" ${DRIVE}

# this makes the ROOT partition
echo "Setup Root"
sgdisk -n 3:0:0 -t 3:8300 -c 3:"Linux / Partition" ${DRIVE}

# this sets the ROOT partitions file system to ext4
mkfs.ext4 ${DRIVE}3
{% endhighlight %}

So that might be alot to take in at first so ill give little explaination on how
***sgdisk*** works,
operators im using and what they do,


**Zap all**,
this is used to destroy GPT and MBR  data stuctures
and then exit, basically wipping and formatting the disk

> -Z



**set alignment**,
aligns the start of the partitions to sectors that are multiples of this value,
this allows obtain optimum performance with SSD drives

> -a



**clear**,
Clears out all Partition data

> -o



**typecode**,
change a single partitions typecode,
uses two-byte hexadecimal number

> -t  



**new partition**,
Create a new partition,
you enter a partition number, starting sector, and an ending sector

> -n



**change name**,
changes the GPT name of a partition, this is encoded as a UTF-16,

> -c



Also after the EFI and ROOT partitions i use the command ***mkfs.fs*** which means
**make file system** and then tell it what file system you want to make e.g **ext4** for
stangered partitions like root or if you wanted to make home a seperate partition to root,
only use **vfat** for the EFI partition

#### Finally,

before we move onto the next part we must mount the partions to there relevant locations
as follows,

{% highlight ruby %}
# this mounts the ROOT partition
echo "# Mounting Partitions"
mount ${DRIVE}3 /mnt

# this creates a /boot/efi directory
mkdir -pv /mnt/boot/efi

# this mounts the EFI
mount ${DRIVE}1 /mnt/boot/efi

# this makes the swap
echo "Enable Swap Partition"
mkswap ${DRIVE}2

# this mounts the swap
swapon ${DRIVE}2
{% endhighlight %}





[guide]: https://wiki.archlinux.org/index.php/installation_guide
[gitrepo]: https://help.github.com/articles/create-a-repo/
[atom]: https://atom.io/
[notpad]: https://notepad-plus-plus.org/download/v7.5.7.html
[lsblk]: ../assets/img/lsblk.png
[packages]: https://git.archlinux.org/archiso.git/tree/configs/releng/packages.x86_64
