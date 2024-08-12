# FreeBSD Installation

## Other Guides

A word of warning regarding guides in general: Be careful about
`sysctl.conf` settings. Check what the current default is. Instead of
improving the situation, you might make it worse.

- The Handbook is useful for several things even beyond the installation
but sometimes it is outdated: [Handbook](https://docs.freebsd.org/en/books/handbook)

- Extensive guide for a variety of things by a long term user: [FreeBSD Desktop](https://vermaden.wordpress.com/freebsd-desktop)

- Guide by a FreeBSD contributor and former member of the Core Team for a Dell Latitude 7390: [My new FreeBSD Laptop: Dell Latitude 7390](https://www.daemonology.net/blog/2020-05-22-my-new-FreeBSD-laptop-Dell-7390.html)

- Guide for a Thinkpad T480: [ThinkPad T480 is my new main laptop which runs FreeBSD](https://genneko.github.io/playing-with-bsd/hardware/freebsd-on-thinkpad-t480)

- Useful for setting up the IPFW firewall: [Recommended Steps For New FreeBSD 12.0 Servers](https://www.digitalocean.com/community/tutorials/recommended-steps-for-new-freebsd-12-0-servers)

## Why FreeBSD?

- Mixture of stable and rolling release. <br />
It consists of two components: the Base, which is a minimal operating
system containing everything necessary to get the system up and
online, and the Ports, which contains all the other software. The Base
is updated with normal releases, while Ports has two options: constant
updates or a 3 months schedule with security updates in between. This
means that you can run a outdated but still supported Base while still
running the latest software through Ports. So after an Base update I
usually wait a few weeks before upgrading until all issues are ironed
out, but I still don't encounter any differences regarding the
software stack.

- ZFS integration.  <br />
ZFS is a Copy-on-write filesystem, which means it is cheap and fast to
create snapshots. FreeBSD not only comes with ZFS, but makes it a
first class citizen. It supports so called boot environments, which
are clones (snapshots that can be edited) of the OS itself. The boot
environments appear in the boot loader, so you can boot back into the
old OS. This also makes rolling back updates trivial. Even if the
update deletes all files of the installation, you are be able to get
back to the situation before the update in no time.

- Ports downloads files only. <br />
If you install software through pkg, it just downloads the
corresponding files, but doesn't do anything else, like starting the
corresponding service. This means that if you, for example, install
your favorite desktop environment and reboot, you won't end up in that
desktop environment; you have to actively activate it if you want to
boot into it. This is not to everybody's taste as it means more
effort, but I quite like it. It means that almost everything that
happens on the system is due to an active choice by you.

## Installation

### Which image should I use?

I recommend using one of the images that includes everything so you
don't need an internet connection during the installation. I used the
`dvd1` image.

### You are behind a proxy server and want to use a smaller image.

Boot the live cd, export the proxy server variables in a shell, verify
your internet connection and execute `bsdinstall`.

### You are using Wifi and want to use a smaller image.

Boot the live cd, export set up the wifi, verify your internet
connection and execute `bsdinstall`.

### The installation itself

Almost all choices can be fixed after the installation. Just pick what
makes sense to you. The only exception is the partitioning and the
filesystem, i.e. whether to pick ZFS or UFS. I generally recommend
choosing ZFS.

There is some old advice out there not to use ZFS with small amounts
of memory, and I let that trick me into using UFS some years ago for
an completely underpowered virtual machine. Later, I reinstalled the
VM with ZFS and it didn't observe any adverse effects. Furthermore,
the features that ZFS offers are in my opinion even worth a
performance penalty. So my advice is to choose ZFS unless you have a
reason not to.

## Wifi

I have an Intel Wifi chipset and use the `iwm` driver. To activate it
add something similar to the following to `/etc/rc.conf`:

```
wlans_iwm0="wlan0"
ifconfig_wlan0="WPA DHCP powersave"
create_args_wlan0="country DE regdomain ETSI"
```

Then populate `/etc/wpa_supplicant.conf` with information about the
Wifi's you want to connect to:

```
ctrl_interface=/var/run/wpa_supplicant
eapol_version=2
ap_scan=1
fast_reauth=1
network={
        ssid="Name1"
        psk="psk1"
}
network={
        ssid="Name2"
        psk="psk2"
}
```

The man page `wpa_supplicant.conf` might be useful

If you want to use the `iwlwifi` driver instead, you might have to
blacklist the `iwm` driver. You can directly block the driver modules
from being loaded by adding to `/boot/loader.conf`

```
module_blacklist="iwm if_iwm"
```

Then you can configure `iwlwifi` in `/etc/rc.conf` like this:

```
devmatch_blacklist="iwm if_iwm"
wlans_iwlwifi0="wlan0"
ifconfig_wlan0="WPA DHCP powersave"
create_args_wlan0="country DE regdomain ETSI"
```

## SSH login

If you want to login via ssh, its daemon sshd needs to be running. To
make it start automatically at boot, you add `sshd_enable="YES"` to
`/etc/rc.conf`. If you only want to use it just occasionally, you
start it with `service sshd onestart` and stop it with `service sshd
onestop`.

## Firewall

There are three different firewalls available: `ip`, `ipfw` and
`ipfilter`. Personally, I use `ipfw` as that was easy to set up. I
just added the following to `/etc/rc.conf`:

```
firewall_enable="YES"
firewall_type="workstation"
firewall_quiet="YES"
firewall_myservices="ssh/tcp"
firewall_allowservices="192.168.178.01/24"
```

This uses the `workstation` type firewall. You can read the
differences between the different types directly in the rule file,
which is `/etc/rc.firewall`. The last two lines above allow `ssh`
connection from my home network to come through. So `ssh` access from
outside is blocked.

## Local Unbound

Local Unbound caches DNS requests on your local machine. You can
activate it during boot by executing
`sysrc local_unbound_enable="YES"`.

## Time Synchronization with NTP

To make sure that your system clock is correct, you can synchronize it
over the internet with NTP. You can active the corresponding daemon at
boot time by executing `sysrc ntpd_enable="YES"`.

## Doas

Why? To run commands as root without having to become the `root` user.
As an alternative you could also use `sudo`. I only use it for `pkg`
operations, for the rest I switch to `root`, so the simpler `doas` is
fine for me.

Install with `pkg install doas`. You can copy the
`/usr/local/etc/doas.conf.sample` to `/usr/local/etc/doas.conf` and
start from there. I commented almost everything out and use it only
for `pkg` commands: (replace `USERNAME` with your actual username)
```
permit USERNAME as root cmd pkg
permit nopass USERNAME as root cmd pkg args update
permit nopass USERNAME as root cmd pkg args upgrade
permit nopass USERNAME as root cmd pkg args autoremove
```
This allows the execution of `pkg` commands that require `root`, e.g.
`doas pkg delete doas`. And the `update`, `upgrade` and `autoremove`
commands won't ask for your password.

## Desktop Environment

The FreeBSD Handbook contains instructions for how to install and set
up the various desktop environments.

## Anacron

Why? `cron` executes commands only at the exact time. If your computer is
offline at that moment, it doesn't execute it at a later point.
`anacron`, on the other hand, runs commands once a specified number of
days has passed. Thus, if a job is missed, it will be executed when
the system comes online.

Install with `pkg install anacron`. Follow the instructions displayed
after installation. (You can see them again `pkg info -D anacron`.)

It boils down to commenting the lines with `periodic` in
`/etc/crontab` and adding a line with
```
0     0       *       *       *       root    /usr/local/sbin/anacron
```
Activate the `anacron` service with `sysrc anacron_enable=YES` and if
you want anacron to wait with starting a new job until all previous
ones are finished, also execute `sysrc anacron_flags+=" -s"`. (I did
that just as a precaution.)

Then add to `/usr/local/etc/anacrontab`:
```
1       5       daily           periodic daily
7       10      weekly          periodic weekly
30      60      monthly         periodic monthly
```

If you want to be notified if updates are available for your system
also add
```
1       15       freebsdupdate   /usr/sbin/freebsd-update cron
```

Note: By default, `periodic` sends the reports to the local mail
acount of `root`. Make sure to check those mails peridically (pun
intended) or set up some forwarding.

## Mail Forwarding

The cron output is send via mail to the root user. If you don't want
to check that, you can forward the mail. I just forward it to my local
user and read it in the terminal with the `mail` command. This
forwarding is set up by editing `/etc/mail/aliases`, e.g. adding

```
root: myuser
```

Afterwards you need to run `newaliases` to update
`/etc/mail/aliases.db` to make it stick.

## /etc/periodic.conf

This file controls the behaviour of the `periodic` jobs. You shouldn't
modify `/etc/defaults/periodic.conf` as that file gets updated during
system updates. Instead you should overwrite the corresponding
variables in `/etc/periodic.conf`. This avoids merge conflicts during
the update.

It might make sense to go through the `/etc/defaults/periodic.conf` to
see what variables you want to change.

Some periodic jobs require an internet connection. If you are behind a
proxy, these jobs might fail despite a functional connection. To work
around this issue, you can just export the proxy in
`/etc/periodic.conf`.

`periodic` jobs sleep by default a random amount of time so that
multiple servers started simultaneously don't all access the internet
at the same time. If you just have one machine, this behaviour makes
no sense. (In fact, on a desktop this might mean that the jobs are not
even started until you shut the machine down again.) To disable that
just include

```
anticongestion_sleeptime=0
```

To cleanup your temporary files you can set

```
daily_clean_tmps_enable="YES"                           # Delete stuff daily
daily_clean_tmps_dirs="/tmp /var/tmp"
```

This deletes files in `daily_clean_tmps_dirs` after they weren't
accessed for `daily_clean_tmps_days`, but excludes files matching the
`daily_clean_tmps_ignore` pattern. The latter two variables I keep
unchanged from their values in `/etc/defaults/periodic.conf`.

If you use ZFS, you might want to enable scrubbing, which verifies
that the stored checksums of the filesystem are correct. To do that
every 14 days, set

```
daily_scrub_zfs_enable="YES"
daily_scrub_zfs_default_threshold="14"                  # days between scrubs
```

You might also want to include in your daily `periodic` jobs a check
of the health of your zpools with

```
daily_status_zfs_enable="YES"                           # Check ZFS
```

## /etc/newsyslog.conf

This file controls how long messages logged via newsyslog are kept,
how they are stored and when the logs are rotated. You might want to
adjust that according to your needs.

## Microcode

CPU microcode updates are available for AMD and Intel CPUs. You need
to install the corresponding package. As I have an Intel CPU, I did
that with `pkg install cpu-microcode-intel`. The installation should
display a message describing how to perform the updates. In general,
there are to ways: 1) the loader performs the updates or 2) a RC script
does it. I do the former, which works for Intel by adding the
following to `/boot/loader.conf`:

```
cpu_microcode_load="YES"
cpu_microcode_name="/boot/firmware/intel-ucode.bin"
```

## Boot Faster

You can improve the time to boot the system by shortening the duration
for which the splash screen is displayed. I don't set it to zero as I
still want to be able to select the single user mode or other boot
environments in the case that the default won't boot. You change the
duration by changing the `autoboot_delay` variable in
`/boot/loader.conf`. For example,

```
autoboot_delay=4
```

changes the duration to 4 seconds.

Another thing that helps the boot time is to disable waiting for USB
devices before mounting `/`. (That makes only sense if `/` is on an
USB device.) You can do that by adding the following to
`/boot/loader.conf`:

```
hsw.usb.no_boot_wait=1
```

## Powersaving

I used three resources to improve the power consumption of my laptop:

 - [FreeBSD wiki](https://wiki.freebsd.org/TuningPowerConsumption)
 - [Vermaden's guide](https://vermaden.wordpress.com/2018/11/28/the-power-to-serve-freebsd-power-management)
 - [Guide for setting up the Intel Speed Shift](https://www.neelc.org/posts/freebsd-speed-shift-laptop)

## Control the Screen Brightness via Keypresses

I originally got the method from copied the guide for the Dell
Latitude 7390, but had to adapt it in the meantime as the
`intel-backlight` package got deprecated. It's successor is FreeBSD's
own `backlight`, which is installed by default. What I had to do on my
Latitude is add the following to `/boot/loader.conf`:

```
acpi_video_load="YES
```

and then create a `/usr/local/etc/devd/acpi-video-backlight.conf` with
the following content:

```
notify 100 {
        match "system" "ACPI";
        match "subsystem" "Video";
        match "type" "brightness";
        action "/usr/bin/backlight $notify";
};
```

## Suspending when Closing the Lid

Add the following to `/boot/loader.conf`:

```
hw.acpi.lid_switch_state=S3
```

## Disable the Bell in the Console

Add the following to `/boot/loader.conf`:

```
kern.vt.enable_bell=0
```

## Allow Users to Mount Stuff

Add the following to `/boot/loader.conf`:

```
vfs.usermount=1
```

## Disable Mouse Buttons 8-10 in Virtualbox

In VirtualBox I had Problems that using the scrollwheel would trigger
history forward and backward events. It seems that every Xth event was
interpreted as a different event, mapped to the history navigation.
You can fix this by disabling the buttons 8 - 10 in `.Xmadmap` with:

```
pointer = 1 2 3 4 5 6 7 0 0 0
```

