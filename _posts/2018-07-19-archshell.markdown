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
PKG="base base-devel grub wireless_tools nfs-utils ntfs-3g openssh pkgfile pacman-contrib mlocate mlocate alsa-utils bash-completion rsync"
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

Notice how the ROOT and EFI partitions are mounted to ***/mnt***, this is short for ***mount***,
this allows you to use the tools on the existing system to setup the file structure for the new system before using it.
Also you have to manually create the ***/mnt/boot/efi*** as seen above.

## 4. Step Four.

#### Downloading packages and installing,

First we will install [**reflector**][reflector], this will allow us to download from the most efficient mirrors for the fastest download,

{% highlight ruby %}
# ths installs reflector
echo "Downloading and Install reflector installation requirements"
pacman -Sy --noconfirm --needed reflector

# this downloads and sort Mirrors List from Archlinux.org
echo "Downloading and Ranking mirrors"
reflector --verbose --protocol http --latest 200 --number 20 --sort rate --save /etc/pacman.d/mirrorlist

# this updates the database
pacman -Syy
{% endhighlight %}

Also in ArchLinux the default package manager is [**Pacman**][pacman] as seen above.
Now its time to install the main system,

{% highlight ruby %}
# this installs all the packages that we have in our PKG variable
echo "# Installing Main System"
pacstrap /mnt ${PKG}

{% endhighlight %}

The most important package to install is the [***base***][base], we have that in the PKG variable up above.
Now lets create the [***Fstab***][fstab], which is what tell arch what drives to mount when it starts up,
installing ***fstab*** is done by the following,

{% highlight ruby %}
# this imstalls the Fstab
echo "# Creating Fstab Entrys"
genfstab -U /mnt >> /mnt/etc/fstab
{% endhighlight %}

## 5. Step Five.

#### Final Configuration for **base.sh**,

To finish off the **base.sh** script we must do some configurations and then [***arch-chroot***][chroot] (this means change root) to the new root
ROOT directory we created, But now we must configure the network settings,

{% highlight ruby %}
# this configures the network
echo "Configuring Network"
rm /mnt/etc/resolv.conf
ln -sf "/run/systemd/resolve/stub-resolv.conf" /mnt/etc/resolv.conf
cat > /mnt/etc/systemd/network/20-wired.network <<NET_EOF
[Match]
Name=en*
[Network]
DHCP=ipv4
NET_EOF
{% endhighlight %}

Now we must set a console keymap by adding youe KEYMAP settings to ***/mnt/etc/vconsole.conf***,

{% highlight ruby %}
# this sets the console keymap
echo "Setting KEYMAP"
echo "KEYMAP=$KEYMAP" >> /mnt/etc/vconsole.conf
{% endhighlight %}

We can set the hostname now by adding the HOSTNAME variable to the ***/mnt/etc/hostname***,

{% highlight ruby %}
# this sets Hostname
echo "Setting Hostname"
echo "${HOSTNAME}" > /mnt/etc/hostname
{% endhighlight %}

To set the loaction to your own country you will have to follow my example, it come defaultly set to US
but im my example i show how to change it to ireland, its the same for everyother country, im using a editor called
***sed*** in this and using the operator ***-i*** to insert text,

{% highlight ruby %}
# this sets the location to ireland
echo "Setting Locale to en_IE"

# this changes the locale to
sed -i 's/^en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
sed -i 's/^#en_IE.UTF-8/en_IE.UTF-8/' /etc/locale.gen

# this sets the language to english
echo "LANG=en_IE.UTF-8" > /etc/locale.conf

export LANG=en_IE.UTF-8
locale-gen
echo ""
{% endhighlight %}

To learn more about locale go [here][locale]
Now to set the time zone, this example used ireland again,

{% highlight ruby %}
# this sets Timezone
echo "Setting Timezone"
ln -sf "/usr/share/zoneinfo/Europe/Dublin" /mnt/etc/localtime
{% endhighlight %}

Next i will enable the network services with [***systemctl***][sysctl] to that it starts on startup,

{% highlight ruby %}
# this enables required services for the network
echo "Setting up Systemd Services"
arch-chroot /mnt systemctl enable systemd-networkd.service systemd-resolved.service
{% endhighlight %}

We must also install [***grub***][grub] with the following,

{% highlight ruby %}
# this installs grub
arch-chroot /mnt grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=Archlinux
# this configures grub
arch-chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg
{% endhighlight %}

Lets now go a head and add in the [***arch-chroot***][chroot] and make it run **post.sh** and we must aslso copy these files over to the new
ROOT,

{% highlight ruby %}
# this copys the post.sh to the new ROOT and runs it
cp post.sh /mnt/root/
arch-chroot /mnt sh /root/post.sh ${ROOTPASSWORD} ${USERNAME} ${USERPASSWORD}
rm /mnt/root/post.sh
{% endhighlight %}

In the **post.sh** we will be configuring the root password and username and password so thats why we are carrying these
variables over.
Now to finish off the **base.sh** we must unmount our drives and give a finnishing note to tell the user that they are to
restart there pc.

{% highlight ruby %}
# this unmounts all the mounted drives
echo "Unmounting Drive Partitions"

# this unmounts the Swap
swapoff ${DRIVE}2

# this unmounts the EFI
umount /mnt/boot/efi

# this unmounts the ROOT
umount /mnt
{% endhighlight %}

And now for a simple finishing note,

{% highlight ruby %}
echo ""
echo "Finised Core Install"
echo
echo
echo
echo "After reboot login as your user"
{% endhighlight %}

Finally, we have the **base.sh** script finished.

## 6. Step Six.

#### Configuration on the post.sh,

Now open your **post.sh** that you have make earlyier and aslready initialised,
first we must take in the variables we sent to it in the **base.sh**, make sure to have them at the
same order that your put them in, like this,

{% highlight ruby %}
# this gives the script the variables
ROOTPASSWORD=$1
USERNAME=$2
USERPASSWORD=$3
{% endhighlight %}

Lets do some updating,

{% highlight ruby %}
# this basically updates all the following
echo "updating"
locale-gen
hwclock --systohc
pacman-key --populate archlinux
pacman-key --init
updatedb
pkgfile --update
{% endhighlight %}

Add this to give a splash of colour,

{% highlight ruby %}
# this adds colour
echo "adding colour"
sed -i 's/#Color/Color/' /etc/pacman.conf
{% endhighlight %}

Now we will set up the user details and passwords for both user and root,

{% highlight ruby %}
# this sets up the root password
echo "creating Root password"
echo -e "${ROOTPASSWORD}\n${ROOTPASSWORD}" | passwd root

# this sets up the user
useradd -m -G wheel,users -s /bin/bash ${USERNAME}

# this sets up the user password
echo -e "${USERPASSWORD}\n${USERPASSWORD}" | passwd ${USERNAME}

# this adds the the user to group for sudo
echo '%wheel ALL=(ALL) ALL' > /etc/sudoers.d/10_wheel
chmod 640 /etc/sudoers.d/10_wheel
{% endhighlight %}

Add this for creating missing directorys,

{% highlight ruby %}
# this creates any missing directories
mkdir -p /etc/pacman.d/hooks
{% endhighlight %}

Now lets give user sudo access,

{% highlight ruby %}
# Change sudoers to allow nobody user access to sudo without password
echo 'nobody ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/99_nobody
{% endhighlight %}

Next build directorys and set there permissions, the [***chmod***][chmod] is used to set permissions,
the [***chgrp***][chgrp] is used to change the group and ***setfacl*** modifies the access control list (**ACL**),the
operator ***-m*** modifies the ACL specified by the **EntryOrFile**,

{% highlight ruby %}
# this creates directorys and set permissions
mkdir /tmp/build
chgrp nobody /tmp/build
chmod g+ws /tmp/build
setfacl -m u::rwx,g::rwx /tmp/build
setfacl -d --set u::rwx,g::rwx,o::- /tmp/build
cd /tmp/build/
{% endhighlight %}

Now lets create the [**AUR**][aur] helper, I will be using [***yay***][yay], this is written in [***GO***][go],
if you install this it can be your one and only packet manager.

{% highlight ruby %}
# Install Yay AUR Helper
sudo -u nobody curl -SLO https://aur.archlinux.org/cgit/aur.git/snapshot/yay.tar.gz
sudo -u nobody tar -zxvf yay.tar.gz
cd yay
sudo -u nobody makepkg -s -i --noconfirm
cd ../..
rm -r build
{% endhighlight %}

Finally we will edit the ***sudoers.d*** again and allow wheel group access to sudo with password,

{% highlight ruby %}
# Change sudoers to allow wheel group access to sudo with password
rm /etc/sudoers.d/99_nobody
{% endhighlight %}

Yay!! we have just finished creating the scripts needed to install ArchLinux,
now just upload it to github 



[guide]: https://wiki.archlinux.org/index.php/installation_guide
[gitrepo]: https://help.github.com/articles/create-a-repo/
[atom]: https://atom.io/
[notpad]: https://notepad-plus-plus.org/download/v7.5.7.html
[lsblk]: ../assets/img/lsblk.png
[packages]: https://git.archlinux.org/archiso.git/tree/configs/releng/packages.x86_64
[reflector]: https://wiki.archlinux.org/index.php/reflector
[pacman]: https://wiki.archlinux.org/index.php/pacman
[base]: https://www.archlinux.org/groups/x86_64/base/
[fstab]: https://wiki.archlinux.org/index.php/fstab
[chroot]: https://wiki.archlinux.org/index.php/Change_root
[locale]: https://wiki.archlinux.org/index.php/Locale
[sysctl]: https://wiki.archlinux.org/index.php/systemd
[grub]: https://wiki.archlinux.org/index.php/GRUB
[chmod]: https://wiki.archlinux.org/index.php/File_permissions_and_attributes#Changing_permissions
[chgrp]: https://www.computerhope.com/unix/uchgrp.html
[aur]: https://aur.archlinux.org/
[yay]: https://github.com/Jguer/yay
[go]: https://golang.org/
