# ThinkPad-FreeBSD-setup

Customized FreeBSD 11 for Lenovo ThinkPad T520. Work in progress.
Please have a look at the list of
[open issues for this repository](https://github.com/eriknstr/ThinkPad-FreeBSD-setup/issues),
as well as the lists of
[open issues for ThinkPad-FreeBSD-ports](https://github.com/eriknstr/ThinkPad-FreeBSD-ports/issues)
and
[open issues for ThinkPad-FreeBSD-src](https://github.com/eriknstr/ThinkPad-FreeBSD-src/issues),
but do also be aware that even if none are open, there could be things
that are broken or incomplete still.

This guide assumes that your Lenovo ThinkPad T520 has a minimum of 8GB RAM,
and that it has a single storage drive; an SSD of at least 120GB in size.
Note that this is different from the factory configuration which has
4GB RAM and a 7200-rpm 500GB HDD. The assumption of the storage medium
being an SSD and not a HDD, in particular, is important, since it will
affect choices about how the OS is to treat the drive. Things which
make sense for a HDD might be slow for a SSD and vice versa.
Things which optimize the usage of a HDD might hurt an SSD and vice versa.

All commands in this README are to be performed as root unless otherwise noted.

![Screenshot of my desktop](https://raw.githubusercontent.com/eriknstr/freebsd-config/screenshots/screenshot.png)

## Table of Contents

* [Install FreeBSD 11](#install-freebsd-11)
  + [ThinkPad T520 BIOS Setup Utility](#thinkpad-t520-bios-setup-utility)
  + [Boot from the FreeBSD 11.0-RC1 install media](#boot-from-the-freebsd-110-rc1-install-media)
  + [Initial post-install configuration](#initial-post-install-configuration)
* [Install custom configuration files](#install-custom-configuration-files)
* [Compile customized system from source](#compile-customized-system-from-source)
  + [Compare changes local to ThinkPad-FreeBSD-src](#compare-changes-local-to-thinkpad-freebsd-src)
  + [Compare changes upstream not yet in ThinkPad-FreeBSD-src](#compare-changes-upstream-not-yet-in-thinkpad-freebsd-src)
* [Custom package builds using Poudriere](#custom-package-builds-using-poudriere)
  + [Installing the packages](#installing-the-packages)
  + [Updating Poudriere ports tree and packages](#updating-poudriere-ports-tree-and-packages)
  + [Keeping an eye on Poudriere builds from other computers](#keeping-an-eye-on-poudriere-builds-from-other-computers)
  + [Compare changes local to ThinkPad-FreeBSD-ports](#compare-changes-local-to-thinkpad-freebsd-ports)
  + [Compare changes upstream not yet in ThinkPad-FreeBSD-ports](#compare-changes-upstream-not-yet-in-thinkpad-freebsd-ports)

## Install FreeBSD 11

Download the official FreeBSD 11.0-RC1 memory stick image [FreeBSD-11.0-RC1-amd64-memstick.img.xz](http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/amd64/ISO-IMAGES/11.0/FreeBSD-11.0-RC1-amd64-memstick.img.xz) and write it on a USB memory stick. Replace `/dev/xxx` below with the path of the path to the block device for the physical "drive" of your USB memory stick.

```bash
sudo umount /dev/xxx*

xzcat ~/Downloads/FreeBSD-11.0-RC1-amd64-memstick.img.xz \
  | sudo dd bs=16M of=/dev/xxx

sudo sync
```

Unplug the USB memory stick, plug it into your ThinkPad T520 and power on
the machine. Press the blue "ThinkVantage"-button, causing it to show
a menu of options (it might also beep at this point).

### ThinkPad T520 BIOS Setup Utility

Press F1 to enter the BIOS Setup Utility.

Navigate to the *Security* menu using the left and right arrow keys
of your keyborad. Select *Virtualization*. Ensure that
*Intel (R) Virtualization Technology* is set to *Enabled* and that
*Intel (R) VT-d Feature* is set to *Enabled*. Press Esc on your
keyboard to accept and go back to the previous menu. Under
*Memory Protection*, ensure *Execution Prevention* is *Enabled*.
Press Esc on your keyboard to go back to *Security* main menu.

TODO: Make use of the Security Chip. Google FreeBSD ThinkPad Security Chip.

Navigate to *Startup*. *Ensure UEFI/Legacy Boot* is set to *Both*
and that *UEFI/Legacy Boot Priority* is set to *UEFI First*.
Ensure *Boot device List F12 Option* is set to *Enabled*.
Next, go into the *Boot* section and ensure that the
*Boot Priority Order* list includes *USB FDD*, *USB HDD*
and *USB CD* somewhere in the list, as well as
*ATA HDD0* and *ATAPI CD0*. Any of these not in the list
must be moved up from *Excluded from boot priority order*.

There are a bunch of other settings worth checking out as well
but I won't list all of them here.

Press F10 to Save and Exit.

Press the blue "ThinkVantage"-button again.

Press F12 to select temporary startup device.

### Boot from the FreeBSD 11.0-RC1 install media

From the Boot Menu, select your USB memory stick.
In my case, it is called "USB HDD".

At the "Welcome to FreeBSD"-screen, press Enter or just wait.

Next, it'll say "Welcome to FreeBSD! Would you like to begin
an installation or use the live CD?" Select *Install* and press
Enter.

Make your *Keymap Selection*. I use the *United States of America dvorak*
keyboard layout (and so should you :). Use the up and down keyboard arrows
to go through the list and press Enter on the one you want to use.
Press Enter again to test the keymap. Type in some letters, numbers
and symbols. Then press Enter, press Enter again to select it and then
use the up keyboard arrow to continue with the keymap you have selected.

Next, enter the hostname you want to use for your computer.
I think one is supposed to enter only the hostname here,
not the fully qualified domain name. I enter just the hostname.

Next it's time to select which optional system components to install.
Select *doc* and unselect *ports*. We'll be using my custom ports tree later.

Next it'll tell us that "Your model of Lenovo is known to have a BIOS
bug that prevents it booting from GPT partitions without UEFI. Would you
like the installer to apply a workaround for you?" Actually, most of all
we would like to boot using UEFI, so just say *No* here, I guess.

Now it's time to partition our disk. We'll select *Auto (ZFS)*, aka.
*Guided Root-on-ZFS*. It will tell us the choices it has made for us
and most of it is good. It has selected not to encrypt disks, which is
fine with me. It says that the *Partition Scheme* will be *GPT (BIOS+UEFI)*,
but we don't want that -- we change the *Partition Scheme* to *GPT (UEFI)*.
For *Swap Size*, we change it from *2g* to *8g* to match our amount of RAM.
All of the rest looks good for now, so we select *Install* to proceed
with the installation.

For *ZFS Configuration* of *Virtual Device type*, we select *stripe*,
since we only have one storage drive.

Next for *ZFS Configuration* it will ask for the devices to use.
Here we select our SSD. For me, it is named *ada0*. Use space to
select it and then press Enter to OK it.

Finally, it will give us one last chance to change our mind.
Why would we? Go ahead and destroy the current contents of
our selected disks. YES!

Then we wait for a while while it partitions the drive
and installs base, kernel, doc and lib32.

Next, enter a password for root. Well, our disk is unencrypted
anyway, so pick something simple like *hest123* and press Enter,
I guess.  (*Hest* is the Norwegian word for horse.)

Enter the password followed by Enter again.

Next, it wants us to configure a network interface.
I usually just go with a wired connection during installs,
so we select the "em0" interface.

Would we like to configure IPv4 for this interface. Yes.

Would we like to use DHCP to configure this interface? Sure.

It acquires DHCP lease.

Would we like to configure IPv6 for this interface? Well,
my current network does not support IPv6 but we'll say yes anyway.

Would we like to try stateless address autoconfiguration (SLAAC)?
Ok. It won't work on this network which does not support IPv6, though.

*Network Configuration* -- *Resolver Configuration* is next.
Since this is a laptop and will thus be on different networks
at various times, we're not going to bother with entering
any search domains. We just press Enter.

Is this machine's CMOS clock set to UTC? Yes, it is,
or at least it will be, so, yes.

Select a region. Probably you don't live the same place I do,
since the country I live in is so small. I select *Europe*
and then *Norway*.

It will ask if some time zone abbreviation looks reasonable.
For me at the current time of the year, this is *CEST*.
Yes, it looks reasonable.

Next, *Time & Date*, date part. The date is already set correctly
so I press Enter to accept that without changing it.

Next, *Time & Date*, time part. My clock is off by a couple of hours
so I attempted to choose *set time* but I guess I should have entered
a different value for it first. Oh well, I'll set it later.

Now it's time to choose the services we would like to be started
at boot. The default selection is *sshd* (good) and *dumpdev*.
I never inspected kernel crash dumps yet, so it is tempting to
uncheck that, but then again, if I ever need to have a look at it
in the future, the benefit of it being enabled so I don't have to
try and replicate the issue outweighs the cost of the disk space,
so we'll leave that selected as well. I am tempted to enable
the local caching validating resolver *local_unbound*. Sometimes
it can be a bit of a pain having one while changing DNS settings
for some domain but I think it's worth it, so I'll select that as well.
Next we'll select *ntpd* to synchronize system and network time
and I'll select *powerd* to adjust CPU frequency dynamically,
hoping that my hardware is supported. We leave *moused* unchecked
and then press Enter to continue.

TODO: Investigate powerd support for my CPU.

Next up is the security hardening options. The ones we'll enable are:

 * *Randomize the PID of newly created processes*
 * *Insert stack guard page ahead of the growable segments*
 * *Clean the /tmp filesystem on system startup*
 * *Disable Sendmail service*

The remainder of the options seem to add nothing for a personal laptop.
As for our disabling the Sendmail service, that's actually just
because we want to install Postfix instead, not for security.

Select the options above using space and press Enter to continue.

Would we like to add users to the installed system now? Yes!

We type in the username. I call my user *erikn*.

We enter our full name. Mine is Erik Nordstrøm, but I'll enter Erik Nordstroem.

When prompted for a Uid, we'll just leave that empty for default.

Login group is named the same as your user, that's what we want.

Would we like to invite our user into other groups? Yes, we'll enter *wheel*.

Login class default is fine.

Our shell can be sh for now. One of the first things we'll do
post-install is to install bash and change our shell to that.

Home directory default is fine as well.

Home directory permissions default is fine.

Use password-based authentication? Yes.

Use an empty password? No.

Use a random password? No.

Enter password. Pick something decent.

Enter the same password again.

Lock out the account after creation? No.

OK? Yes.

Add another user? No.

*Final configuration*. *Exit* -- apply configuration and exit installer.

Wait a little while for the configuration to be applied.

*Manual configuration*. "The installation is now finished. Before exiting
the installer, would you like to open a shell in the new system to make
any final manual modifications?" No, we'll take care of the rest on the
live system after reboot.

Installation of FreeBSD complete! Would we like to reboot
into the installed system now? Yes, please :)

### Initial post-install configuration

The system reboots. Log in as `root` using the password you selected.

We are eventually going to overwrite `/etc/rc.conf` but would like
to preserve the options we selected during bsdinstall. In order to do this,
we move our current `/etc/rc.conf` to `/etc/rc.conf.local`.

```sh
mv /etc/rc.conf /etc/rc.conf.local
```

Snapshot your freshly installed system. Download and run the script
https://raw.githubusercontent.com/eriknstr/utils/master/snap.sh
or just run `sh` and type in the following manually:

```sh
snapname="$( date +%Y-%m-%d )-$( freebsd-version )-$( date +%s )"

zfs snapshot -r zroot@$snapname

zfs destroy -r zroot/tmp@$snapname
zfs destroy -r zroot/var/log@$snapname
zfs destroy -r zroot/var/crash@$snapname
zfs destroy -r zroot/usr/ports@$snapname

exit
```

## Install custom configuration files

TODO: Maintain system configs in repo ThinkPad-FreeBSD-src on branch stable/11.

TODO: Maintain most package options in repo ThinkPad-FreeBSD-ports on branch master.

```sh
pkg bootstrap

pkg install git

mkdir -p /root/src/github.com/eriknstr/

cd /root/src/github.com/eriknstr/

git clone -b stable/11 git@github.com:eriknstr/ThinkPad-FreeBSD-setup.git

cd ThinkPad-FreeBSD-setup

./install.sh
```

Configure WLAN. Most of it is taken care of by the installed files,
but you'll need to enter information about network SSID and PSK.
If your SSID was *mysweetwifi* and your PSK was *supersecret*,
then you'd add the following contents to the file `/etc/wpa_supplicant.conf`:

```
network={
	ssid="mysweetwifi"
	psk="supersecret"
}
```

Reboot.

## Compile customized system from source

NOTE: At the time of this writing, no changes have yet been made by me to the FreeBSD sources, so there is no point in recompiling from source yet. [Compare changes local to ThinkPad-FreeBSD-src](https://github.com/freebsd/freebsd/compare/stable/11...eriknstr:stable/11).

First time, do the following. Subsequent times, skip this step.

```sh
cd /usr

git clone -b stable/11 git@github.com:eriknstr/ThinkPad-FreeBSD-src.git src

cd src

git remote add upstream git@github.com:freebsd/freebsd.git

ln -s /root/src/github.com/eriknstr/ThinkPad-FreeBSD-setup/zroot/usr/src/sys/amd64/conf/T520 \
  sys/amd64/conf/T520
```

First time, skip this step. Subsequent times, start with this step.
If this step says that you are up to date then there is no point
in rebuilding everything since you'd end up with the same thing
you already had.

```sh
cd /usr/src

git pull
```

Reboot into single user mode, then

```sh
/singleuser.sh

cd /usr/src

make buildworld buildkernel installkernel KERNCONF=T520
```

A full buildworld buildkernel takes about five hours
on my ThinkPad T520 at the time of this writing.
Out of this, the time taken by the buildkernel step
alone is about 50 minutes.

Reboot into single user mode again and then

```sh
/singleuser.sh

cd /usr/src

mergemaster -p

make installworld

mergemaster -iF

make delete-old
```

Finally, reboot into multi user mode and then do

```sh
make delete-old-libs
```

TODO: Package rebuilding in relation to delete-old-libs

See also:

 * https://www.freebsd.org/doc/handbook/kernelconfig-building.html
 * https://www.freebsd.org/doc/handbook/makeworld.html

### Compare changes local to ThinkPad-FreeBSD-src

https://github.com/freebsd/freebsd/compare/stable/11...eriknstr:stable/11

### Compare changes upstream not yet in ThinkPad-FreeBSD-src

https://github.com/eriknstr/ThinkPad-FreeBSD-src/compare/stable/11...freebsd:stable/11

## Custom package builds using Poudriere

TODO: Create jail from customized FreeBSD build
instead of using the official release files.

```sh
pkg install poudriere

poudriere jail -c -j 11amd64 -v 11.0-RC1

cd /usr/local/poudriere/ports/

git clone git@github.com:eriknstr/ThinkPad-FreeBSD-ports.git local

cd local

git remote add upstream git@github.com:freebsd/freebsd-ports.git

poudriere bulk -j 11amd64 -p local -z python35 \
  -f /usr/local/etc/poudriere.d/11amd64-local-python35-pkglist

poudriere bulk -j 11amd64 -p local -z python34 \
  -f /usr/local/etc/poudriere.d/11amd64-local-python34-pkglist

poudriere bulk -j 11amd64 -p local -z python27 \
  -f /usr/local/etc/poudriere.d/11amd64-local-python27-pkglist
```

See also: https://www.freebsd.org/doc/handbook/ports-poudriere.html

### Installing the packages

```sh
cut -d'/' -f2 /usr/local/etc/poudriere.d/11amd64-local-python35-pkglist \
  | xargs -L1 pkg install
```

(We run the install one package at a time so that a missing package
 won't stop the installation of all the other, unrelated packages.)

### Updating Poudriere ports tree and packages

```sh
cd /usr/local/poudriere/ports/local/

git pull

poudriere bulk -j 11amd64 -p local -z python35 \
  -f /usr/local/etc/poudriere.d/11amd64-local-python35-pkglist

poudriere bulk -j 11amd64 -p local -z python34 \
  -f /usr/local/etc/poudriere.d/11amd64-local-python34-pkglist

poudriere bulk -j 11amd64 -p local -z python27 \
  -f /usr/local/etc/poudriere.d/11amd64-local-python27-pkglist
```

### Keeping an eye on Poudriere builds from other computers

```sh
cd /usr/local/poudriere/data/logs/bulk
doas -u www python3.5 -m http.server
```

### Compare changes local to ThinkPad-FreeBSD-ports

https://github.com/freebsd/freebsd-ports/compare/master...eriknstr:master

### Compare changes upstream not yet in ThinkPad-FreeBSD-ports

https://github.com/eriknstr/ThinkPad-FreeBSD-ports/compare/master...freebsd:master
