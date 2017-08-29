---
title: 'Fedora Flock Atomic Host 101 Lab Part 0: Preparation'
author: dustymabe
date: 2017-08-29
tags: [ fedora, atomic ]
published: true
---

# Introduction

While Atomic Host has been around since 2014 there are still a lot of
people that aren't as familiar with the technology. The Atomic team
within Red Hat, along with numerous other upstream contributors, have
brought the [OSTree](https://ostree.readthedocs.io/en/latest/manual/introduction/)
and [RPM-OSTree](https://rpm-ostree.readthedocs.io/en/latest/)
technology a long way. At the Fedora user and contributor conference (known as 
[Flock](https://rpm-ostree.readthedocs.io/en/latest/)) this week we will be
giving a [lab on Atomic Host](https://flock2017.sched.com/event/Bm97/atomic-host-101)
designed to let new users learn about
Atomic Host. The audience for this lab is anyone familiar with Linux
and interested in learning a new technology.

The entire lab will cover quite a few features of Atomic Host
including:

- Configuring Storage for Containers
- Atomic Host Rebasing 
- Atomic Host Upgrades and Rollbacks
- Viewing Changes To A Deployed System
- Browsing OS History
- Package Layering
- Experimental Features (livefs, override, remove)

In *Part 0: Preparation* we are going to set up an existing Fedora 26
Atomic Host for the rest of the lab sections.


# Getting A Fedora 26 Atomic Host

In this lab you'll need to start with a Fedora 26 Atomic Host. It is
recommended you start from the `26.110` (`13ed0f2`) release. The
corresponding media that was built from that release was part of
the `26-20170821.0` compose. You can spin up this release by booting 
in AWS or by spinning up a vagrant box or cloud image. Information on
AMIs or links to downloads can be found [here](content/post/2017-08-29/imageinfo.txt).

For demonstration purposes I'll be using the libvirt vagrant box. If
you are on Fedora please see the [Fedora Developer Portal](https://developer.fedoraproject.org/tools/vagrant/about.html)
for instructions on how to get started with Vagrant.

After downloading the libvirt vagrant box
(`Fedora-Atomic-Vagrant-26-20170821.0.x86_64.vagrant-libvirt.box`)
we can now add the box to Vagrant: 

```nohighlight
[user@laptop ~]#$ vagrant box add --name F26AHLab ./Fedora-Atomic-Vagrant-26-20170821.0.x86_64.vagrant-libvirt.box
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'F26AHLab' (v0) for provider:
    box: Unpacking necessary files from: file:///path/Fedora-Atomic-Vagrant-26-20170821.0.x86_64.vagrant-libvirt.box
==> box: Successfully added box 'F26AHLab' (v0) for 'libvirt'!
```

I can now bring up and ssh into the Atomic Host:

```
$ mkdir atomic && cd atomic && vagrant init F26AHLab && vagrant up
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
Bringing machine 'default' up with 'libvirt' provider...
==> default: Creating image (snapshot of base box volume).
...
$ vagrant ssh
[vagrant@localhost ~]$
[vagrant@localhost ~]$
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]# grep PRETTY_NAME /etc/os-release
PRETTY_NAME="Fedora 26 (Atomic Host)"
[root@localhost ~]#
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
● fedora-atomic:fedora/26/x86_64/atomic-host
                   Version: 26.110 (2017-08-20 18:10:09)
                    Commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424
```


# Extend Root Filesystem

For this lab we are going to be adding some files to the root
filesystem and also rebasing to other OSTrees. Let's add a significant
amount of space to the root filesystem to accommodate this:

```
[root@localhost ~]# lvresize --resizefs --size=+10G /dev/atomicos/root
  Size of logical volume atomicos/root changed from 2.93 GiB (750 extents) to 12.93 GiB (3310 extents).
  Logical volume atomicos/root successfully resized.
...
``` 

# Download Lab Files Archive

A goal of this lab is to be able to be executed in an offline
scenario. This means we have archived a bunch of files of the lab
and stored them in a tar.gz for consumption. It also means that a
user should be able to execute this lab any time in the future without
worrying about missing necessary files that are no longer on the web.

Please download the [atomic-host-lab.tar.gz](https://s3.amazonaws.com/atomic-host-lab/atomic-host-lab.tar.gz)
file into the `/srv/localweb` directory on the running atomic host:

**NOTE:** If you downloaded the `atomic-host-lab.tar.gz` file before
          you started the Atomic Host then you can use `scp` to copy
          the file in.

```nohighlight
[root@localhost ~]# mkdir /srv/localweb && chmod 777 /srv/localweb
[root@localhost ~]# cd /src/localweb
[root@localhost localweb]# curl -O https://s3.amazonaws.com/atomic-host-lab/atomic-host-lab.tar.gz
...
[root@localhost localweb]# tar -xf atomic-host-lab.tar.gz
[root@localhost localweb]# rm atomic-host-lab.tar.gz
[root@localhost localweb]# cd
[root@localhost ~]#
```

# Set Up Local Webserver For Archive Files

Throughout this entire lab we'll use a local webserver to host
the files from the archive. Initially this service will use builtin
http server modules within Python. Let's create and start that
service now:


```nohighlight
[root@localhost ~]# cat <<'EOF' > /etc/systemd/system/localweb.service
[Unit]

[Service]
WorkingDirectory=/srv/localweb
ExecStart=/usr/bin/python3 -m http.server

[Install]
WantedBy=multi-user.target
EOF
[root@localhost ~]#
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl enable --now localweb.service
Created symlink /etc/systemd/system/multi-user.target.wants/localweb.service → /etc/systemd/system/localweb.service.
[root@localhost ~]# systemctl status localweb.service
● localweb.service
   Loaded: loaded (/etc/systemd/system/localweb.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2017-08-29 16:36:45 UTC; 5s ago
 Main PID: 2974 (python3)
    Tasks: 1 (limit: 4915)
   Memory: 19.6M
      CPU: 342ms
   CGroup: /system.slice/localweb.service
           └─2974 /usr/bin/python3 -m http.server

Aug 29 16:36:45 localhost.localdomain systemd[1]: Started localweb.service.
[root@localhost ~]#
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world
```


# Configure OSTree/YUM Repositories

Our local webservice is now running and actually hosting files that
are a front for an OSTree repository as well as a YUM repository.
Let's remove the old remote from the system and configure it so
that `ostree` pulls from that local server:


```nohighlight
[root@localhost ~]# ostree remote delete fedora-atomic
[root@localhost ~]# ostree remote add --no-gpg-verify local http://localhost:8000/ostreerepo/
```

Let's take it one step farther and tell the current deployment
on the system to trcak the `fedora/26/x86_64/updates/atomic-host` 
ref from the `local` remote.

```nohighlight
[root@localhost ~]# ostree admin set-origin --index 0 local http://localhost:8000/ostreerepo/ fedora/26/x86_64/updates/atomic-host
```

Finally, we'll delete all existing YUM repositories and add our
`localyum` repository to the system.

```nohighlight
[root@localhost ~]# rm /etc/yum.repos.d/*.repo
[root@localhost ~]# cat <<'EOF' > /etc/yum.repos.d/localyum.repo
[localyum]
baseurl=http://localhost:8000/yumrepo/
enabled=1
gpgcheck=0
EOF
```

# Part 0 Wrap Up

Part 0 of this lab has been specifically to get the system prepared
for later sections of the lab. You may have some questions. That's OK.
Hopefully we'll answer them in the later sections of this lab.




# investigate atomic host: 

underlying technology known as ostree and leveraged by an rpm aware
higher level technology known as rpm-ostree. rpm-ostree is able to 
build and deliver ostrees built out of rpms (yay!)

[root@vanilla-f26atomic ~]# rpm-ostree status
State: idle
Deployments:
● fedora-atomic:fedora/26/x86_64/atomic-host
                   Version: 26.110 (2017-08-20 18:10:09)
                    Commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424

# how does this differ from a normal system? content is tracked by
# the rpm-ostree software. For example, for this commit you can 
# see all of the rpms that are installed in this tree:


[root@localhost ~]# rpm-ostree db list 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424 | head
ostree commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424
 GeoIP-1.6.11-1.fc26.x86_64
 GeoIP-GeoLite-data-2017.07-1.fc26.noarch
 NetworkManager-1:1.8.2-1.fc26.x86_64
 NetworkManager-libnm-1:1.8.2-1.fc26.x86_64
 NetworkManager-team-1:1.8.2-1.fc26.x86_64
 acl-2.2.52-15.fc26.x86_64
 atomic-1.18.1-5.fc26.x86_64
 atomic-devmode-0.3.7-1.fc26.noarch
 atomic-registries-1.18.1-5.fc26.x86_64


# One nice feature of this system is that rpm queries still work

[root@localhost ~]# rpm -q kernel
kernel-4.12.5-300.fc26.x86_64

# However, typical rpm installation does not

[root@localhost ~]# rpm -ivh /srv/localweb/yumrepo/htop-2.0.2-2.fc26.x86_64.rpm
warning: /srv/localweb/yumrepo/htop-2.0.2-2.fc26.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 64dab85d: NOKEY
error: can't create transaction lock on /var/lib/rpm/.rpm.lock (No such file or directory)
[root@localhost ~]#
[root@localhost ~]# dnf install /srv/localweb/yumrepo/htop-2.0.2-2.fc26.x86_64.rpm
bash: dnf: command not found

**NOTE:** Even though you can't install RPMs directly using traditional tools you can
          add rpms via package layering. More on this later.

# why can't we install rpms? mostly because content is static
(read/only) content. Let's look at the filesystem structure? 

[root@localhost ~]# ls -l /
total 18
lrwxrwxrwx.   2 root root    7 Aug 21 09:00 bin -> usr/bin
drwxr-xr-x.   8 root root 1024 Aug 21 09:02 boot
drwxr-xr-x.  19 root root 3380 Aug 28 00:15 dev
drwxr-xr-x.  81 root root 8192 Aug 28 00:23 etc
lrwxrwxrwx.   2 root root    8 Aug 21 09:00 home -> var/home
lrwxrwxrwx.   3 root root    7 Aug 21 09:00 lib -> usr/lib
lrwxrwxrwx.   3 root root    9 Aug 21 09:00 lib64 -> usr/lib64
lrwxrwxrwx.   2 root root    9 Aug 21 09:00 media -> run/media
lrwxrwxrwx.   2 root root    7 Aug 21 09:00 mnt -> var/mnt
lrwxrwxrwx.   2 root root    7 Aug 21 09:00 opt -> var/opt
lrwxrwxrwx.   2 root root   14 Aug 21 09:01 ostree -> sysroot/ostree
dr-xr-xr-x. 114 root root    0 Aug 28 00:15 proc
lrwxrwxrwx.   2 root root   12 Aug 21 09:01 root -> var/roothome
drwxr-xr-x.  35 root root 1080 Aug 28 00:15 run
lrwxrwxrwx.   2 root root    8 Aug 21 09:01 sbin -> usr/sbin
lrwxrwxrwx.   2 root root    7 Aug 21 09:01 srv -> var/srv
dr-xr-xr-x.  13 root root    0 Aug 28 00:15 sys
drwxr-xr-x.  11 root root  112 Aug 21 09:00 sysroot
lrwxrwxrwx.   2 root root   11 Aug 21 09:01 tmp -> sysroot/tmp
drwxr-xr-x.  12 root root  155 Jan  1  1970 usr
drwxr-xr-x.  24 root root 4096 Aug 28 00:15 var


# alot of stateful directories point to /var/.
# alot of non stateful directories point to /usr/

[root@localhost ~]# touch /var/foofile
[root@localhost ~]# ls -l /var/foofile
-rw-r--r--. 1 root root 0 Aug 28 00:33 /var/foofile
[root@localhost ~]# touch foo /usr/file
touch: cannot touch '/usr/file': Read-only file system

# var is read/write. and a lot of usr is readonly

# except usr/local and usr/tmp

[root@localhost ~]# ls -l /usr/local /usr/tmp
lrwxrwxrwx. 2 root root 15 Aug 21 09:01 /usr/local -> ../var/usrlocal
lrwxrwxrwx. 2 root root 10 Aug 21 09:01 /usr/tmp -> ../var/tmp

# configuration in etc is also read/write 
# and also tracked. you can 'diff' between what was provided with the
# ostree and what exists on the system 

# ostree admin config-diff | head
M    machine-id
M    subgid
M    subuid
M    hosts
M    localtime
M    systemd/logind.conf
M    systemd/system/default.target
M    group
M    shadow
M    passwd
...

# The 'M' means Modified. An 'A' would mean added, while a 'D' would
mean the file had been deleted.

# Let's play around with one of the files:

[root@localhost ~]# cat /etc/motd
[root@localhost ~]# ostree admin config-diff | grep motd

# no change in motd from what was delivered with the system
We'll add some state and then check again

[root@localhost ~]# echo 'Fedora Atomic Host is Awesome!' >> /etc/motd
[root@localhost ~]# ostree admin config-diff | grep motd
M    motd

# But what changed? Currently with `ostree admin config-diff` we can only see if
# it was modified added or deleted. We can however dig into the
# deployments and see the real diff:

[root@localhost ~]# diff -ur /ostree/deploy/fedora-atomic/deploy/13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0/usr/etc/motd /etc/motd
--- /ostree/deploy/fedora-atomic/deploy/13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0/usr/etc/motd 1970-01-01 00:00:00.000000000 +0000
+++ /etc/motd   2017-08-28 00:37:01.003412786 +0000
@@ -0,0 +1 @@
+Fedora Atomic Host is Awesome!

# log out and log back in and see the new message of the day!

[root@localhost ~]# exit
logout
[vagrant@localhost ~]$ exit
logout
Connection to 192.168.121.57 closed.
[user@laptop]$ vagrant ssh
Fedora Atomic Host is Awesome!
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]#



# Later we'll see that this state is also tracked between the various deployments on a system and will allow us to rollback to previous deployments and also rollback the state in /etc/




# Container storage 

blah blah blah - container storage 


# investigate docker-storage-setup options
cat /etc/sysconfig/docker-storage-setup

# did they work? 

[root@localhost ~]# systemctl status docker-storage-setup.service -o cat
● docker-storage-setup.service - Docker Storage Setup
   Loaded: loaded (/usr/lib/systemd/system/docker-storage-setup.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since Mon 2017-08-28 00:15:50 UTC; 27min ago
 Main PID: 720 (code=exited, status=0/SUCCESS)

Starting Docker Storage Setup...
CHANGED: partition=2 start=616448 old: size=83269632 end=83886080 new: size=85366784,end=85983232
  Physical volume "/dev/vda2" changed
  1 physical volume(s) resized / 0 physical volume(s) not resized
  Logical volume "docker-root-lv" created.
Started Docker Storage Setup.
[root@localhost ~]# docker info 2>/dev/null | grep 'Storage Driver'
Storage Driver: overlay2
[root@localhost ~]# lsblk
NAME                          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda                           252:0    0   41G  0 disk
├─vda1                        252:1    0  300M  0 part /boot
└─vda2                        252:2    0 40.7G  0 part
  ├─atomicos-root             253:0    0   13G  0 lvm  /sysroot
  └─atomicos-docker--root--lv 253:1    0 15.1G  0 lvm  /sysroot/ostree/deploy/fedora-atomic/var/lib/docker

# how do we switch things up? 

If you want to change the container storage I suggest you use the
`container-storage-setup` utility, previously known as `docker-storage-setup`.
Look at the `/usr/share/container-storage-setup/container-storage-setup` file
for an explanation of options that you can put in the
`/etc/sysconfig/docker-storage-setup` configuration file.

For example, if you want to switch from `overlay2` to `devicemapper`
you would need to change it so that `STORAGE_DRIVER=devicemapper`.
As an exercise, let's switch this system to the devicemapper backend:


We'll first stop docker, unmount the filesystem which was backing
the overlay2 driver and also remove the logical volume that filesystem
was build on top of.

[root@localhost ~]# systemctl stop docker
[root@localhost ~]#
[root@localhost ~]# umount /var/lib/docker
[root@localhost ~]# lvremove /dev/atomicos/docker-root-lv
Do you really want to remove active logical volume atomicos/docker-root-lv? [y/n]: y
  Logical volume "docker-root-lv" successfully removed


Next we'll overwrite remove the `/etc/sysconfig/docker-storage`, which
has options to give to the docker daemon to tell it what storage
driver to us, and also we'll overwrite all settings in the 
`/etc/sysconfig/docker-storage-setup` so that `container-storage-setup` 
will know we want to use the `devicemapper` backend. We'll then run
`container-storage-setup` to create the devicemapper thin pool 
and populate a new `/etc/sysconfig/docker-storage` with options for
the docker daemon.

[root@localhost ~]# rm /etc/sysconfig/docker-storage
[root@localhost ~]# echo 'STORAGE_DRIVER=devicemapper' > /etc/sysconfig/docker-storage-setup
[root@localhost ~]# /usr/bin/container-storage-setup
  Using default stripesize 64.00 KiB.
  Rounding up size to full physical extent 44.00 MiB
  Logical volume "docker-pool" created.
  Logical volume atomicos/docker-pool changed.


Now that this is done you can see thin LV that was set up in the'
`lsblk` and `lvs` output:

[root@localhost ~]# lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda                             252:0    0   41G  0 disk
├─vda1                          252:1    0  300M  0 part /boot
└─vda2                          252:2    0 40.7G  0 part
  ├─atomicos-root               253:0    0   13G  0 lvm  /sysroot
  ├─atomicos-docker--pool_tmeta 253:1    0   44M  0 lvm
  │ └─atomicos-docker--pool     253:3    0   11G  0 lvm
  └─atomicos-docker--pool_tdata 253:2    0   11G  0 lvm
    └─atomicos-docker--pool     253:3    0   11G  0 lvm
[root@localhost ~]# lvs
  LV          VG       Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  docker-pool atomicos twi-a-t--- 11.02g             0.00   0.09
  root        atomicos -wi-ao---- 12.93g

Now we can start docker and verify that the devicemapper backend is
being used:

[root@localhost ~]# systemctl start docker
[root@localhost ~]# docker info 2>/dev/null | grep 'Storage Driver'
Storage Driver: devicemapper

To cap things off let's import a container image and run a container
to see that new devicemapper objects are getting created for this
storage:

[root@localhost ~]# atomic storage import --dir  /srv/localweb/containers/
Importing image: a16c8800bb14
7d4769f4070d: Loading layer [==================================================>] 242.5 MB/242.5 MB
25ca5f0393cd: Loading layer [==================================================>]   359 MB/359 MB
960139190bea: Loading layer [==================================================>] 71.77 MB/71.77 MB
Loaded image: registry.fedoraproject.org/f25/httpd:latest
Importing volumes
atomic import completed successfully
Would you like to cleanup (rm -rf /srv/localweb/containers/) the temporary directory [y/N]N
Please restart docker daemon for the changes to take effect
[root@localhost ~]#
[root@localhost ~]# docker run -d a16c8800bb14 sleep 600
b0509ea6c6b6811915f3f2d15a9fd344836f628a638d797683e7b27c4be84205
[root@localhost ~]#
[root@localhost ~]# lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda                             252:0    0   41G  0 disk
├─vda1                          252:1    0  300M  0 part /boot
└─vda2                          252:2    0 40.7G  0 part
  ├─atomicos-root               253:0    0   13G  0 lvm  /sysroot
  ├─atomicos-docker--pool_tmeta 253:1    0   44M  0 lvm
  │ └─atomicos-docker--pool     253:3    0   11G  0 lvm
  │   └─docker-253:0-14918141-d748db4190b91666509effaabaa06c24d6e5ed836e5421ce914250e7eb1ac5a0
  │                             253:4    0   10G  0 dm   /var/lib/docker/devicemapper/mnt/d748db4190b916
  └─atomicos-docker--pool_tdata 253:2    0   11G  0 lvm
    └─atomicos-docker--pool     253:3    0   11G  0 lvm
      └─docker-253:0-14918141-d748db4190b91666509effaabaa06c24d6e5ed836e5421ce914250e7eb1ac5a0
                                253:4    0   10G  0 dm   /var/lib/docker/devicemapper/mnt/d748db4190b916


We can see the new devicemapper mounts that were created for the
container. 

# rebasing?

One of the more fascinating facets of Atomic Host techology is that you can rebase to completely different operating system trees and back again. Let's do the unthinkable. Go from the newer technology in Fedora to the older technology in CentOS. You can achieve this by `rebasing` to the centos ostree.


rpm-ostree rebase local:centos-atomic-host/7/x86_64/standard

    [root@localhost ~]# rpm-ostree rebase local:centos-atomic-host/7/x86_64/standard

    1908 metadata, 14284 content objects fetched; 414666 KiB transferred in 84 seconds
    Copying /etc changes: 23 modified, 0 removed, 63 added
    Transaction complete; bootconfig swap: yes deployment count change: 1
    ...
      kernel 4.12.5-300.fc26 -> 3.10.0-514.26.2.el7
    ...
    Run "systemctl reboot" to start a reboot
    [root@localhost ~]# reboot



    [user@laptop ~]$ vagrant ssh
	Last login: Mon Aug 28 00:37:32 2017 from 192.168.121.1
	Fedora Atomic Host is Awesome!
	[vagrant@localhost ~]$ 
	[vagrant@localhost ~]$ sudo su -
	[root@localhost ~]# cat /etc/redhat-release 
	CentOS Linux release 7.3.1611 (Core) 
	[root@localhost ~]# 
	[root@localhost ~]# rpm-ostree status
	State: idle
	Deployments:
	● local:centos-atomic-host/7/x86_64/standard
				 Version: 7.1707 (2017-07-31 16:12:06)
				  Commit: 0bf6200211dd4fd63be6e9bc5c90bea645e2696c0117b05f83562081813a5b94

	  local:fedora/26/x86_64/updates/atomic-host
				 Version: 26.110 (2017-08-20 18:10:09)
				  Commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424

**NOTE:** The ● indicates the currently booted deployment and the order of the
          status output indicates the boot priority order.


# pretty amazing. We made it from Fedora 26 to CentOS 7 Atomic Host. 
# One of the first things to notice is that now our MOTD is Wrong!
# Let's Change it to mention CentOS:

	[root@localhost ~]# echo 'CentOS Atomic Host is Awesome!' > /etc/motd
	[root@localhost ~]# exit
	logout
	[vagrant@localhost ~]$ exit
	logout
	Connection to 192.168.121.57 closed.
	[user@laptop ~]$ vagrant ssh
	Last login: Mon Aug 28 01:09:18 2017 from 192.168.121.1
	CentOS Atomic Host is Awesome!
	[vagrant@localhost ~]$ sudo su -
	[root@localhost ~]# 

# Let's poke around to see what else is awry:

[root@localhost ~]# systemctl --failed | head -n 4
  UNIT             LOAD   ACTIVE SUB    DESCRIPTION
● kdump.service    loaded failed failed Crash recovery kernel arming
● localweb.service loaded failed failed localweb.service

# Any guesses as to why localweb failed to start? I'll give you hint: Python 3. 

# Fedora includes Python 3 by default where CentOS doesn't have it included 
# yet. Our service is using Python 3 and a Python 3 only module. We need to fix it
# by replacing `python3 -m http.server` with `python -m SimpleHTTPServer`.

[root@localhost ~]# sed -i 's|/usr/bin/python3 -m http.server|/usr/bin/python -m SimpleHTTPServer|'  /etc/systemd/system/localweb.service
[root@localhost ~]# systemctl daemon-reload && systemctl start localweb
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world


# Let's check docker to see if it is running fine? 

[root@localhost ~]# docker info | grep 'Storage Driver' 
Storage Driver: devicemapper
[root@localhost ~]# docker images
REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
registry.fedoraproject.org/f25/httpd   latest              a16c8800bb14        3 days ago          648.7 MB
[root@localhost ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
b0509ea6c6b6        a16c8800bb14        "sleep 600"         16 minutes ago      Exited (0) 6 minutes ago                       vibrant_lalande

Looks good. Still using devicemapper as the storage backend. The container 
image we imported is still there and we can even see the container we started 
before we did the rebase. 

While this trip to neverland (Fedora 26 -> CentOS 7) was a fun exercise
let's head back to where we were before we started this adventure. First
let's take a moment to review the changes we made to the system since we
booted CentOS 7.

- we've updated the motd
- we've fixed the localweb systemd unit 

# At this point you can actually see the diff in these files between the old 
# deployment and the current deployment 

[root@localhost ~]# diff -ur /ostree/deploy/fedora-atomic/deploy/13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0/etc/motd /etc/motd
--- /ostree/deploy/fedora-atomic/deploy/13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0/etc/motd     2017-08-28 00:37:01.003412786 +0000
+++ /etc/motd   2017-08-28 01:11:51.099930383 +0000
@@ -1 +1 @@
-Fedora Atomic Host is Awesome!
+CentOS Atomic Host is Awesome!
[root@localhost ~]# 
[root@localhost ~]# diff -ur /ostree/deploy/fedora-atomic/deploy/13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0/etc/systemd/system/localweb.service /etc/systemd/system/localweb.service
--- /ostree/deploy/fedora-atomic/deploy/13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0/etc/systemd/system/localweb.service  2017-08-28 00:24:57.228412786 +0000
+++ /etc/systemd/system/localweb.service        2017-08-28 01:14:12.367930383 +0000
@@ -2,7 +2,7 @@
 
 [Service]
 WorkingDirectory=/srv/localweb
-ExecStart=/usr/bin/python3 -m http.server
+ExecStart=/usr/bin/python -m SimpleHTTPServer
 
 [Install]
 WantedBy=multi-user.target


# Keep these changes in mind as we do the rollback to Fedora 26.

[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
● local:centos-atomic-host/7/x86_64/standard
             Version: 7.1707 (2017-07-31 16:12:06)
              Commit: 0bf6200211dd4fd63be6e9bc5c90bea645e2696c0117b05f83562081813a5b94

  local:fedora/26/x86_64/updates/atomic-host
             Version: 26.110 (2017-08-20 18:10:09)
              Commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424
[root@localhost ~]# rpm-ostree rollback --reboot 
Moving '13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424.0' to be first deployment
Transaction complete; bootconfig swap: yes deployment count change: 0
Connection to 192.168.121.57 closed by remote host.
Connection to 192.168.121.57 closed.


# back to fedora


$ vagrant ssh 
Last login: Mon Aug 28 01:12:04 2017 from 192.168.121.1
Fedora Atomic Host is Awesome!
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]#

# first thing you will notice is we are back to Fedora is awesome message
# the rollback reverted all the configuration changes in /etc/

Let's check the other, less obvious, file that we had changed that should
have been reverted:

[root@localhost ~]# systemctl cat localweb.service
# /etc/systemd/system/localweb.service
[Unit]

[Service]
WorkingDirectory=/srv/localweb
ExecStart=/usr/bin/python3 -m http.server

[Install]
WantedBy=multi-user.target


As expected, we're back on Python 3. Is the localweb service working?

[root@localhost ~]# 
[root@localhost ~]# systemctl status localweb
● localweb.service
   Loaded: loaded (/etc/systemd/system/localweb.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2017-08-28 01:23:25 UTC; 1min 51s ago
 Main PID: 669 (python3)
    Tasks: 1 (limit: 4915)
   Memory: 19.2M
      CPU: 332ms
   CGroup: /system.slice/localweb.service
           └─669 /usr/bin/python3 -m http.server

Aug 28 01:23:25 localhost.localdomain systemd[1]: Started localweb.service.
[root@localhost ~]#
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world

Looks good!

# THis means we have rolled back fully to our pre-upgrade state
# The status output shows us we are back to fedora as our booted deployment:

[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.110 (2017-08-20 18:10:09)
                    Commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424

  local:centos-atomic-host/7/x86_64/standard
                   Version: 7.1707 (2017-07-31 16:12:06)
                    Commit: 0bf6200211dd4fd63be6e9bc5c90bea645e2696c0117b05f83562081813a5b94


# Instead of rebasing to CentOS, like we did earlier, what should we have been doing?
# Most likely just checing to see if an upgrade existing for our existing ref we are
# following. Let's do that now:


[root@localhost ~]# rpm-ostree upgrade 

810 metadata, 3678 content objects fetched; 174260 KiB transferred in 21 seconds                                                                                                                                  
Copying /etc changes: 23 modified, 4 removed, 70 added
Transaction complete; bootconfig swap: yes deployment count change: 0
Freed objects: 24.8 MB
Freed pkgcache branches: 1 size: 246.6 kB
Upgraded:
  bind99-libs 9.9.10-1.P2.fc26 -> 9.9.10-2.P3.fc26
  bind99-license 9.9.10-1.P2.fc26 -> 9.9.10-2.P3.fc26
  ca-certificates 2017.2.14-1.0.fc26 -> 2017.2.16-1.0.fc26
  coreutils 8.27-5.fc26 -> 8.27-6.fc26
  coreutils-common 8.27-5.fc26 -> 8.27-6.fc26
  expat 2.2.3-1.fc26 -> 2.2.4-1.fc26
  file 5.30-9.fc26 -> 5.30-10.fc26
  file-libs 5.30-9.fc26 -> 5.30-10.fc26
  glibc 2.25-8.fc26 -> 2.25-9.fc26
  glibc-all-langpacks 2.25-8.fc26 -> 2.25-9.fc26
  glibc-common 2.25-8.fc26 -> 2.25-9.fc26
  kernel 4.12.5-300.fc26 -> 4.12.8-300.fc26
  kernel-core 4.12.5-300.fc26 -> 4.12.8-300.fc26
  kernel-modules 4.12.5-300.fc26 -> 4.12.8-300.fc26
  libcrypt-nss 2.25-8.fc26 -> 2.25-9.fc26
  librepo 1.7.20-3.fc26 -> 1.8.0-1.fc26
  lz4-libs 1.7.5-4.fc26 -> 1.8.0-1.fc26
  nspr 4.15.0-1.fc26 -> 4.16.0-1.fc26
  nss 3.31.0-1.1.fc26 -> 3.32.0-1.1.fc26
  nss-softokn 3.31.0-1.0.fc26 -> 3.32.0-1.2.fc26
  nss-softokn-freebl 3.31.0-1.0.fc26 -> 3.32.0-1.2.fc26
  nss-sysinit 3.31.0-1.1.fc26 -> 3.32.0-1.1.fc26
  nss-tools 3.31.0-1.1.fc26 -> 3.32.0-1.1.fc26
  nss-util 3.31.0-1.0.fc26 -> 3.32.0-1.0.fc26
  oci-umount 2:1.13.1-21.git27e468e.fc26 -> 2:2.0.0-2.gitf90b64c.fc26
  ostree 2017.9-2.fc26 -> 2017.10-2.fc26
  ostree-grub2 2017.9-2.fc26 -> 2017.10-2.fc26
  ostree-libs 2017.9-2.fc26 -> 2017.10-2.fc26
  p11-kit 0.23.5-3.fc26 -> 0.23.8-1.fc26
  p11-kit-trust 0.23.5-3.fc26 -> 0.23.8-1.fc26
  python3-rpm 4.13.0.1-5.fc26 -> 4.13.0.1-7.fc26
  rpm 4.13.0.1-5.fc26 -> 4.13.0.1-7.fc26
  rpm-build-libs 4.13.0.1-5.fc26 -> 4.13.0.1-7.fc26
  rpm-libs 4.13.0.1-5.fc26 -> 4.13.0.1-7.fc26
  rpm-ostree 2017.7-1.fc26 -> 2017.8-2.fc26
  rpm-plugin-selinux 4.13.0.1-5.fc26 -> 4.13.0.1-7.fc26
  sqlite-libs 3.20.0-1.fc26 -> 3.20.0-2.fc26
  vim-minimal 2:8.0.946-1.fc26 -> 2:8.0.983-1.fc26
Added:
  rpm-ostree-libs-2017.8-2.fc26.x86_64
Run "systemctl reboot" to start a reboot
[root@localhost ~]# reboot 
Connection to 192.168.121.57 closed by remote host.
Connection to 192.168.121.57 closed.
$ vagrant ssh 
Last login: Mon Aug 28 02:05:28 2017 from 192.168.121.1
Fedora Atomic Host is Awesome!
[vagrant@localhost ~]$ 
[vagrant@localhost ~]$ 
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                    Commit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5

  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.110 (2017-08-20 18:10:09)
                    Commit: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424


# Browsing histroy

The nice think about OSTree being like git for your OS is that you can browse the history:
What happened in the commits betwee `26.110` and `26.115`? Let's find out:



[root@localhost ~]# rpm-ostree db diff 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424 a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5 
ostree diff commit old: 13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424
ostree diff commit new: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
Upgraded:
  bind99-libs 9.9.10-1.P2.fc26.x86_64 -> 9.9.10-2.P3.fc26.x86_64
  bind99-license 9.9.10-1.P2.fc26.noarch -> 9.9.10-2.P3.fc26.noarch
  ca-certificates 2017.2.14-1.0.fc26.noarch -> 2017.2.16-1.0.fc26.noarch
  coreutils 8.27-5.fc26.x86_64 -> 8.27-6.fc26.x86_64
  coreutils-common 8.27-5.fc26.x86_64 -> 8.27-6.fc26.x86_64
  expat 2.2.3-1.fc26.x86_64 -> 2.2.4-1.fc26.x86_64
  file 5.30-9.fc26.x86_64 -> 5.30-10.fc26.x86_64
  file-libs 5.30-9.fc26.x86_64 -> 5.30-10.fc26.x86_64
  glibc 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  glibc-all-langpacks 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  glibc-common 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  kernel 4.12.5-300.fc26.x86_64 -> 4.12.8-300.fc26.x86_64
  kernel-core 4.12.5-300.fc26.x86_64 -> 4.12.8-300.fc26.x86_64
  kernel-modules 4.12.5-300.fc26.x86_64 -> 4.12.8-300.fc26.x86_64
  libcrypt-nss 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  librepo 1.7.20-3.fc26.x86_64 -> 1.8.0-1.fc26.x86_64
  lz4-libs 1.7.5-4.fc26.x86_64 -> 1.8.0-1.fc26.x86_64
  nspr 4.15.0-1.fc26.x86_64 -> 4.16.0-1.fc26.x86_64
  nss 3.31.0-1.1.fc26.x86_64 -> 3.32.0-1.1.fc26.x86_64
  nss-softokn 3.31.0-1.0.fc26.x86_64 -> 3.32.0-1.2.fc26.x86_64
  nss-softokn-freebl 3.31.0-1.0.fc26.x86_64 -> 3.32.0-1.2.fc26.x86_64
  nss-sysinit 3.31.0-1.1.fc26.x86_64 -> 3.32.0-1.1.fc26.x86_64
  nss-tools 3.31.0-1.1.fc26.x86_64 -> 3.32.0-1.1.fc26.x86_64
  nss-util 3.31.0-1.0.fc26.x86_64 -> 3.32.0-1.0.fc26.x86_64
  oci-umount 2:1.13.1-21.git27e468e.fc26.x86_64 -> 2:2.0.0-2.gitf90b64c.fc26.x86_64
  ostree 2017.9-2.fc26.x86_64 -> 2017.10-2.fc26.x86_64
  ostree-grub2 2017.9-2.fc26.x86_64 -> 2017.10-2.fc26.x86_64
  ostree-libs 2017.9-2.fc26.x86_64 -> 2017.10-2.fc26.x86_64
  p11-kit 0.23.5-3.fc26.x86_64 -> 0.23.8-1.fc26.x86_64
  p11-kit-trust 0.23.5-3.fc26.x86_64 -> 0.23.8-1.fc26.x86_64
  python3-rpm 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm-build-libs 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm-libs 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm-ostree 2017.7-1.fc26.x86_64 -> 2017.8-2.fc26.x86_64
  rpm-plugin-selinux 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  sqlite-libs 3.20.0-1.fc26.x86_64 -> 3.20.0-2.fc26.x86_64
  vim-minimal 2:8.0.946-1.fc26.x86_64 -> 2:8.0.983-1.fc26.x86_64
Added:
  rpm-ostree-libs-2017.8-2.fc26.x86_64




[root@localhost ~]# old=''; commit='a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5'
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree db diff "${commit}${old}^" "${commit}${old}"; old+='^'
ostree diff commit old: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^ (59bc8e66abe22c4338aecbd300b5343f0e44537204496dc25f0541b079b28b4d)
ostree diff commit new: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
Upgraded:
  librepo 1.7.20-3.fc26.x86_64 -> 1.8.0-1.fc26.x86_64
  oci-umount 2:1.13.1-21.git27e468e.fc26.x86_64 -> 2:2.0.0-2.gitf90b64c.fc26.x86_64
  python3-rpm 4.13.0.1-6.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm 4.13.0.1-6.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm-build-libs 4.13.0.1-6.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm-libs 4.13.0.1-6.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
  rpm-plugin-selinux 4.13.0.1-6.fc26.x86_64 -> 4.13.0.1-7.fc26.x86_64
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree db diff "${commit}${old}^" "${commit}${old}"; old+='^'
ostree diff commit old: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^ (7c1f406901ef991dd556b1a69b008f63346f4fcf831dccdf9cb115c76385a246)
ostree diff commit new: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^ (59bc8e66abe22c4338aecbd300b5343f0e44537204496dc25f0541b079b28b4d)
Upgraded:
  bind99-libs 9.9.10-1.P2.fc26.x86_64 -> 9.9.10-2.P3.fc26.x86_64
  bind99-license 9.9.10-1.P2.fc26.noarch -> 9.9.10-2.P3.fc26.noarch
  expat 2.2.3-1.fc26.x86_64 -> 2.2.4-1.fc26.x86_64
  p11-kit 0.23.5-3.fc26.x86_64 -> 0.23.8-1.fc26.x86_64
  p11-kit-trust 0.23.5-3.fc26.x86_64 -> 0.23.8-1.fc26.x86_64
  sqlite-libs 3.20.0-1.fc26.x86_64 -> 3.20.0-2.fc26.x86_64
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree db diff "${commit}${old}^" "${commit}${old}"; old+='^'
ostree diff commit old: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^^ (4c57eeb2478c6a0a4c41c3ec2fa900e8258725a84cc3e87f510ce271c1774dc6)
ostree diff commit new: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^ (7c1f406901ef991dd556b1a69b008f63346f4fcf831dccdf9cb115c76385a246)
Upgraded:
  coreutils 8.27-5.fc26.x86_64 -> 8.27-6.fc26.x86_64
  coreutils-common 8.27-5.fc26.x86_64 -> 8.27-6.fc26.x86_64
  kernel 4.12.5-300.fc26.x86_64 -> 4.12.8-300.fc26.x86_64
  kernel-core 4.12.5-300.fc26.x86_64 -> 4.12.8-300.fc26.x86_64
  kernel-modules 4.12.5-300.fc26.x86_64 -> 4.12.8-300.fc26.x86_64
  nspr 4.15.0-1.fc26.x86_64 -> 4.16.0-1.fc26.x86_64
  nss 3.31.0-1.1.fc26.x86_64 -> 3.32.0-1.1.fc26.x86_64
  nss-softokn 3.31.0-1.0.fc26.x86_64 -> 3.32.0-1.2.fc26.x86_64
  nss-softokn-freebl 3.31.0-1.0.fc26.x86_64 -> 3.32.0-1.2.fc26.x86_64
  nss-sysinit 3.31.0-1.1.fc26.x86_64 -> 3.32.0-1.1.fc26.x86_64
  nss-tools 3.31.0-1.1.fc26.x86_64 -> 3.32.0-1.1.fc26.x86_64
  nss-util 3.31.0-1.0.fc26.x86_64 -> 3.32.0-1.0.fc26.x86_64
  ostree 2017.9-2.fc26.x86_64 -> 2017.10-2.fc26.x86_64
  ostree-grub2 2017.9-2.fc26.x86_64 -> 2017.10-2.fc26.x86_64
  ostree-libs 2017.9-2.fc26.x86_64 -> 2017.10-2.fc26.x86_64
  rpm-ostree 2017.7-1.fc26.x86_64 -> 2017.8-2.fc26.x86_64
  vim-minimal 2:8.0.946-1.fc26.x86_64 -> 2:8.0.983-1.fc26.x86_64
Added:
  rpm-ostree-libs-2017.8-2.fc26.x86_64
[root@localhost ~]# 
[root@localhost ~]# 
^[[A[root@localhost ~]# rpm-ostree db diff "${commit}${old}^" "${commit}${old}"; old+^C
[root@localhost ~]# rpm-ostree db diff "${commit}${old}^" "${commit}${old}"; old+='^'
ostree diff commit old: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^^^ (55be41d9a8f5b7d774652aa4a4d407d496b8892d3d51a6585e6ee268940677ad)
ostree diff commit new: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^^ (4c57eeb2478c6a0a4c41c3ec2fa900e8258725a84cc3e87f510ce271c1774dc6)
Upgraded:
  ca-certificates 2017.2.14-1.0.fc26.noarch -> 2017.2.16-1.0.fc26.noarch
  glibc 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  glibc-all-langpacks 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  glibc-common 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  libcrypt-nss 2.25-8.fc26.x86_64 -> 2.25-9.fc26.x86_64
  lz4-libs 1.7.5-4.fc26.x86_64 -> 1.8.0-1.fc26.x86_64
  python3-rpm 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-6.fc26.x86_64
  rpm 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-6.fc26.x86_64
  rpm-build-libs 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-6.fc26.x86_64
  rpm-libs 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-6.fc26.x86_64
  rpm-plugin-selinux 4.13.0.1-5.fc26.x86_64 -> 4.13.0.1-6.fc26.x86_64
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree db diff "${commit}${old}^" "${commit}${old}"; old+='^'
ostree diff commit old: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^^^^ (13ed0f241c9945fd5253689ccd081b5478e5841a71909020e719437bbeb74424)
ostree diff commit new: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5^^^^ (55be41d9a8f5b7d774652aa4a4d407d496b8892d3d51a6585e6ee268940677ad)
Upgraded:
  file 5.30-9.fc26.x86_64 -> 5.30-10.fc26.x86_64
  file-libs 5.30-9.fc26.x86_64 -> 5.30-10.fc26.x86_64

# 

Now let's mutate the ostree a bit by layering in one of my favorite RPMs: `htop`. 


[root@localhost ~]# rpm-ostree install htop 
Checking out tree a8db0b7... done
Enabled rpm-md repositories: localyum
rpm-md repo 'localyum' (cached); generated: 2017-08-27 17:54:44
Importing metadata 5%
Importing metadata 100%
Resolving dependencies... done
Will download: 1 package (106.1 kB)
  Downloading from localyum: 15%
  Downloading from localyum: 30%
  Downloading from localyum: 46%
  Downloading from localyum: 61%
  Downloading from localyum: 77%
  Downloading from localyum: 92%
  Downloading from localyum: 100%
Importing: 100%
Applying 1 overlay... done
Running pre scripts... 0 done
Running post scripts... 3 done
Writing rpmdb... done
Writing OSTree commit... done
Copying /etc changes: 23 modified, 4 removed, 71 added
Transaction complete; bootconfig swap: yes deployment count change: 0
Added:
  htop-2.0.2-2.fc26.x86_64
Run "systemctl reboot" to start a reboot

The status output will now show us the new pending deployment (staged
for next boot) with `htop` now tracked as part of `LayeredPackages`.

[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
           LayeredPackages: htop

● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                    Commit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5


Let's go ahead and reboot and use htop!


	[root@localhost ~]# reboot 
	Connection to 192.168.121.57 closed by remote host.
	Connection to 192.168.121.57 closed.
	$ vagrant ssh 
	Last login: Mon Aug 28 02:08:19 2017 from 192.168.121.1
	Fedora Atomic Host is Awesome!
	[vagrant@localhost ~]$ sudo su -
	[root@localhost ~]# rpm -q htop
	htop-2.0.2-2.fc26.x86_64
	[root@localhost ~]# htop -v
	htop 2.0.2 - (C) 2004-2017 Hisham Muhammad
	Released under the GNU GPL.
	[root@localhost ~]# htop


# experimental features

livefs blah blah


[root@localhost ~]# rpm-ostree install nano | tee 
Checking out tree a8db0b7... done
Enabled rpm-md repositories: localyum
rpm-md repo 'localyum' (cached); generated: 2017-08-28 22:07:11

Importing metadata 5%
Importing metadata 100%
Resolving dependencies... done
Importing: 100%
Applying 2 overlays... done
Running pre scripts... 0 done
Running post scripts... 4 done
Writing rpmdb... done
Writing OSTree commit... done
Copying /etc changes: 23 modified, 4 removed, 72 added
Transaction complete; bootconfig swap: no deployment count change: 0
Added:
  nano-2.8.4-1.fc26.x86_64
Run "systemctl reboot" to start a reboot
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
           LayeredPackages: htop nano

● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
           LayeredPackages: htop
[root@localhost ~]# rpm-ostree ex livefs 
Diff Analysis: 0e12b24e08407d22b71ac971be12dcb6f32b51f4e4e2962960adce0ba4ad07bd => 887f1d79408575a3b99b5c73674f84c3d69fea9b0e6e0e3617d95e296e0bfc82
Files: modified: 1 removed: 0 added: 145
Packages: modified: 0 removed: 0 added: 1
* Configuration changed in /etc
Preparing new rollback matching currently booted deployment
Copying /etc changes: 23 modified, 4 removed, 72 added
Transaction complete; bootconfig swap: yes deployment count change: 1
Overlaying /usr... done
Copying new config files... 1


[root@localhost ~]# rpm -q nano
nano-2.8.4-1.fc26.x86_64

[root@localhost ~]# nano /etc/motd 
[root@localhost ~]# cat /etc/motd
Fedora Atomic Host is Awesome! Edited with nano!


[root@vanilla-f26atomic ~]# rpm -q strace
strace-4.18-1.fc26.x86_64
[root@vanilla-f26atomic ~]# ls /usr/bin/strace
/usr/bin/strace
[root@vanilla-f26atomic ~]# ls -l /usr/bin/strace
-rwxr-xr-x. 4 root root 1146968 Jan  1  1970 /usr/bin/strace

#rpm-ostree ex override remove strace 

[root@localhost ~]# rpm-ostree ex override remove strace
Checking out tree a8db0b7... done
Enabled rpm-md repositories: localyum
rpm-md repo 'localyum' (cached); generated: 2017-08-28 22:07:11

Importing metadata 5%
Importing metadata 100%
Resolving dependencies... done
Importing: 100%
Applying 1 override and 2 overlays... done
Running pre scripts... 0 done
Running post scripts... 4 done
Writing rpmdb... done
Writing OSTree commit... done
Copying /etc changes: 23 modified, 4 removed, 72 added
Transaction complete; bootconfig swap: no deployment count change: 0
Removed:
  strace-4.18-1.fc26.x86_64
Added:
  nano-2.8.4-1.fc26.x86_64
Run "systemctl reboot" to start a reboot
[root@localhost ~]# 

#try to livefs - notice that livefs doesn't work with modified/removed

[root@localhost ~]# rpm-ostree ex livefs 
Note: Previous overlay: 887f1d79408575a3b99b5c73674f84c3d69fea9b0e6e0e3617d95e296e0bfc82
Diff Analysis: 0e12b24e08407d22b71ac971be12dcb6f32b51f4e4e2962960adce0ba4ad07bd => c70d813bacf424948986aae8c918c28295c35524237413df7601c93ded480f22
Files: modified: 1 removed: 2 added: 145
Packages: modified: 0 removed: 1 added: 1
* Configuration changed in /etc
error: livefs update modifies/replaces packages



[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
                    Commit: c70d813bacf424948986aae8c918c28295c35524237413df7601c93ded480f22
       RemovedBasePackages: strace-4.18-1.fc26.x86_64
           LayeredPackages: htop nano

● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
          BootedBaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
                    Commit: 0e12b24e08407d22b71ac971be12dcb6f32b51f4e4e2962960adce0ba4ad07bd
                LiveCommit: 887f1d79408575a3b99b5c73674f84c3d69fea9b0e6e0e3617d95e296e0bfc82
           LayeredPackages: htop

  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
                    Commit: 0e12b24e08407d22b71ac971be12dcb6f32b51f4e4e2962960adce0ba4ad07bd
           LayeredPackages: htop

# reboot

[root@localhost ~]# rpm -q strace
strace-4.18-1.fc26.x86_64
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# reboot 
Connection to 192.168.121.57 closed by remote host.
Connection to 192.168.121.57 closed.
$ 
$ 
$ vagrant ssh 
Last login: Mon Aug 28 02:20:06 2017 from 192.168.121.1

Fedora Atomic Host is Awesome! Edited with nano!
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]# rpm -q strace
package strace is not installed
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
                    Commit: c70d813bacf424948986aae8c918c28295c35524237413df7601c93ded480f22
       RemovedBasePackages: strace-4.18-1.fc26.x86_64
           LayeredPackages: htop nano

  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
          BootedBaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
                    Commit: 0e12b24e08407d22b71ac971be12dcb6f32b51f4e4e2962960adce0ba4ad07bd
                LiveCommit: 887f1d79408575a3b99b5c73674f84c3d69fea9b0e6e0e3617d95e296e0bfc82
           LayeredPackages: htop

  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
                    Commit: 0e12b24e08407d22b71ac971be12dcb6f32b51f4e4e2962960adce0ba4ad07bd
           LayeredPackages: htop



We just added a new package to Atomic Host. Typically we try not to install
many packages this way mostly because keeping the diff small between the OSTree that
was delivered and the OSTree that is running on your system will mean that you
are running a system that is more similar to the artifacts that were tested as 
a unit before they were released. Ideally most of your applications will run as containers. 

Speaking of containers, how about we fire one up? 


The container image we imported earlier was a Fedora 25 httpd container image. Let's
get rid of the Python based localweb server in favor of something a little more powerful.


systemctl disable --now localweb
systemctl status localweb


docker run -d --restart=always --name httpd -p 8000:8080 -v /srv/localweb/:/var/www/html/:Z a16c8800bb14

curl localhost:8000/hello.txt


[root@localhost ~]# systemctl disable --now localweb
Removed /etc/systemd/system/multi-user.target.wants/localweb.service.
[root@localhost ~]# systemctl status localweb
● localweb.service
   Loaded: loaded (/etc/systemd/system/localweb.service; disabled; vendor preset: disabled)
   Active: inactive (dead)

Aug 29 03:35:12 localhost.localdomain systemd[1]: Started localweb.service.
Aug 29 03:39:13 localhost.localdomain systemd[1]: Stopping localweb.service...
Aug 29 03:39:13 localhost.localdomain systemd[1]: Stopped localweb.service.
[root@localhost ~]# 
[root@localhost ~]# docker run -d --restart=always --name httpd -p 8000:8080 -v /srv/localweb/:/var/www/html/:Z a16c8800bb14
03fb1de49dd7af4e517827e1b046647b5c1c8f57a1210e63f43f1ac7e922685f
[root@localhost ~]# 
[root@localhost ~]# docker ps 
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS                                               NAMES
03fb1de49dd7        a16c8800bb14        "/usr/bin/run-httpd"   About a minute ago   Up 54 seconds       80/tcp, 443/tcp, 8443/tcp, 0.0.0.0:8000->8080/tcp   httpd
[root@localhost ~]# curl http://localhost:8000/hello.txt                                                
hello world

# looks good. Let's move on to the next experimental feature where we can actually override
# a package that was delivered with atomic host. This can be useful for testing a new version
# of a package to see if a bug was fixed:
# For example, now that we have a container running we notice that a new version of the docker
# package exists in the testing repositories. We want to test it out to see if the new version
# of the package works. We can use *override replace* to achieve this:



test out new docker: 

	[root@localhost ~]# rpm-ostree ex override replace /srv/localweb/yumrepo/docker*
Checking out tree a8db0b7... done
Enabled rpm-md repositories: localyum
rpm-md repo 'localyum' (cached); generated: 2017-08-28 22:07:11


Importing metadata [==============================================================================] 100%
Resolving dependencies... done
Applying 4 overrides and 2 overlays... done
Running pre scripts... 0 done
Running post scripts... 7 done
Writing rpmdb... done
Writing OSTree commit... done
Copying /etc changes: 23 modified, 4 removed, 72 added
Transaction complete; bootconfig swap: yes deployment count change: -1
Upgraded:
  docker 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26
  docker-common 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26
  docker-rhel-push-plugin 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26
Run "systemctl reboot" to start a reboot
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
       RemovedBasePackages: strace-4.18-1.fc26.x86_64
      ReplacedBasePackages: docker-rhel-push-plugin 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26, docker-common 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26, docker 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26
           LayeredPackages: htop nano

● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
       RemovedBasePackages: strace-4.18-1.fc26.x86_64
           LayeredPackages: htop nano



[root@localhost ~]# reboot 
Connection to 192.168.121.57 closed by remote host.
Connection to 192.168.121.57 closed.
$ 
$ 
$ vagrant ssh 
Last login: Mon Aug 28 02:20:06 2017 from 192.168.121.1
[root@localhost ~]# rpm -q docker
docker-1.13.1-22.gitb5e3294.fc26.x86_64
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                               NAMES
03fb1de49dd7        a16c8800bb14        "/usr/bin/run-httpd"   9 minutes ago       Up About a minute   80/tcp, 443/tcp, 8443/tcp, 0.0.0.0:8000->8080/tcp   httpd
[root@localhost ~]# 
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world



# get rid of all of the overrideds


[root@localhost ~]# rpm-ostree ex override reset strace docker-rhel-push-plugin docker-common docker
Checking out tree a8db0b7... done
Enabled rpm-md repositories: localyum
rpm-md repo 'localyum' (cached); generated: 2017-08-28 22:07:11


Importing metadata [==============================================================================] 100%
Resolving dependencies... done
Applying 2 overlays... done
Running pre scripts... 0 done
Running post scripts... 4 done
Writing rpmdb... done
Writing OSTree commit... done
Copying /etc changes: 23 modified, 4 removed, 73 added
Transaction complete; bootconfig swap: no deployment count change: 0
Downgraded:
  docker 2:1.13.1-22.gitb5e3294.fc26 -> 2:1.13.1-21.git27e468e.fc26
  docker-common 2:1.13.1-22.gitb5e3294.fc26 -> 2:1.13.1-21.git27e468e.fc26
  docker-rhel-push-plugin 2:1.13.1-22.gitb5e3294.fc26 -> 2:1.13.1-21.git27e468e.fc26
Added:
  strace-4.18-1.fc26.x86_64
Run "systemctl reboot" to start a reboot
[root@localhost ~]# 
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
           LayeredPackages: htop nano

● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
       RemovedBasePackages: strace-4.18-1.fc26.x86_64
      ReplacedBasePackages: docker-rhel-push-plugin 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26, docker-common 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26, docker 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26
           LayeredPackages: htop nano
[root@localhost ~]# 
[root@localhost ~]# reboot 
Connection to 192.168.121.57 closed by remote host.
Connection to 192.168.121.57 closed.

$ vagrant ssh 
Last login: Tue Aug 29 03:45:57 2017 from 192.168.121.1
Fedora Atomic Host is Awesome! Edited with nano!
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
           LayeredPackages: htop nano

  local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
       RemovedBasePackages: strace-4.18-1.fc26.x86_64
      ReplacedBasePackages: docker-rhel-push-plugin 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26, docker-common 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26, docker 2:1.13.1-21.git27e468e.fc26 -> 2:1.13.1-22.gitb5e3294.fc26
           LayeredPackages: htop nano
[root@localhost ~]# 
[root@localhost ~]# rpm -q strace docker
strace-4.18-1.fc26.x86_64
docker-1.13.1-21.git27e468e.fc26.x86_64
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                               NAMES
03fb1de49dd7        a16c8800bb14        "/usr/bin/run-httpd"   14 minutes ago      Up 29 seconds       80/tcp, 443/tcp, 8443/tcp, 0.0.0.0:8000->8080/tcp   httpd
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world


# You don't have to use containers to use atomic host

# You can also combine operations. For example, let's rebase to rawhide (because we like to
# live life on the edge) and also install a package at the same time.

[root@localhost ~]# rpm-ostree rebase local:fedora/rawhide/x86_64/atomic-host --install httpd | tee     
2 metadata, 0 content objects fetched; 884 B transferred in 0 seconds
Checking out tree 55a65a6... done
Enabled rpm-md repositories: localyum
rpm-md repo 'localyum' (cached); generated: 2017-08-28 22:07:11

Importing metadata 100%
Resolving dependencies... done
Importing: 100%
Relabeling 2 packages: 100%
Applying 10 overlays... done
Running pre scripts... 1 done
Running post scripts... 6 done
Writing rpmdb... done
Writing OSTree commit... done
Copying /etc changes: 23 modified, 4 removed, 74 added
Transaction complete; bootconfig swap: yes deployment count change: 0
Freed pkgcache branches: 3 size: 77.0 MB
Upgraded:
...
  kernel 4.12.8-300.fc26 -> 4.13.0-0.rc6.git2.1.fc28
...
Added:
  httpd-2.4.27-6.fc27.x86_64
...
Run "systemctl reboot" to start a reboot
[root@localhost ~]# rpm-ostree status
State: idle
Deployments:
  local:fedora/rawhide/x86_64/atomic-host
                   Version: Rawhide.20170824.n.0 (2017-08-24 14:35:23)
                BaseCommit: 55a65a66f736e7637a23ddb9b649546d7b4ea247c35e32f61047dc7882d08a93
           LayeredPackages: htop httpd nano

● local:fedora/26/x86_64/updates/atomic-host
                   Version: 26.115 (2017-08-26 19:46:28)
                BaseCommit: a8db0b7d3f2e54e4092d5ed640087934b8424637cfa1e3ce4bbaf7ccccfd09a5
           LayeredPackages: htop nano

[root@localhost ~]# 
[root@localhost ~]# reboot 
Connection to 192.168.121.57 closed by remote host.
Connection to 192.168.121.57 closed.
$ vagrant ssh 
Last login: Tue Aug 29 03:52:25 2017 from 192.168.121.1
Fedora Atomic Host is Awesome! Edited with nano!
[vagrant@localhost ~]$ 
[vagrant@localhost ~]$ 
[vagrant@localhost ~]$ sudo su -
[root@localhost ~]# 

# First things first, let's add some new info to our MOTD :)
# with our "still layered" nano package:

[root@localhost ~]# nano /etc/motd 
[root@localhost ~]# 
[root@localhost ~]# cat /etc/motd
Fedora Rawhide Atomic Host is Awesome! Edited with nano!

# notice updated packages and the fact that docker container is still running:

[root@localhost ~]# uname -r
4.13.0-0.rc6.git2.1.fc28.x86_64
[root@localhost ~]# rpm -q docker
docker-1.13.1-30.gitb5e3294.fc28.x86_64
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                               NAMES
03fb1de49dd7        a16c8800bb14        "/usr/bin/run-httpd"   22 minutes ago      Up 29 seconds       80/tcp, 443/tcp, 8443/tcp, 0.0.0.0:8000->8080/tcp   httpd
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world


# kill docker container in preparation for httpd serving content

[root@localhost ~]# docker rm -f httpd 
httpd

# XXX open a bug for gpgcheck issues

# switch to using httpd instead of docker to 


chcon -R unconfined_u:object_r:httpd_sys_content_t:s0 /srv/localweb/
rmdir /var/www/html
ln -s /srv/localweb/ /var/www/html

sed -i 's|Listen 80|Listen 8000|' /etc/httpd/conf/httpd.conf
semanage port -m -t http_port_t -p tcp 8000
soundd_port_t

systemctl enable --now httpd
systemctl status httpd

curl localhost:8000/hello.txt


[root@localhost ~]# chcon -R unconfined_u:object_r:httpd_sys_content_t:s0 /srv/localweb/
[root@localhost ~]# rmdir /var/www/html
[root@localhost ~]# ln -s /srv/localweb/ /var/www/html
[root@localhost ~]# sed -i 's|Listen 80|Listen 8000|' /etc/httpd/conf/httpd.conf
[root@localhost ~]# semanage port -m -t http_port_t -p tcp 8000
[root@localhost ~]# systemctl enable --now httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
[root@localhost ~]# systemctl status httpd -o cat
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2017-08-29 04:06:28 UTC; 11s ago
     Docs: man:httpd.service(8)
 Main PID: 1407 (httpd)
   Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
    Tasks: 213 (limit: 4915)
   CGroup: /system.slice/httpd.service
           ├─1407 /usr/sbin/httpd -DFOREGROUND
           ├─1408 /usr/sbin/httpd -DFOREGROUND
           ├─1409 /usr/sbin/httpd -DFOREGROUND
           ├─1410 /usr/sbin/httpd -DFOREGROUND
           └─1412 /usr/sbin/httpd -DFOREGROUND

Starting The Apache HTTP Server...
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.l
Started The Apache HTTP Server.
[root@localhost ~]# 
[root@localhost ~]# curl http://localhost:8000/hello.txt
hello world
