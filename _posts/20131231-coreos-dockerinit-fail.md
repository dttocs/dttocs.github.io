---
layout: post
title: Docker containers fail to start after CoreOS upgrade
description: "After a CoreOS auto-upgrade, no containers would start, failing with a .dockerinit error"
modified: 2013-12-31
tags: [docker coreos]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: false
---

I have been playing around with [Docker](http://www.docker.io/) for a few weeks, and started looking at
[CoreOS](http://coreos.com/) as a way to easily run Docker under VMware Fusion in a lightweight VM.
CoreOS has a built in [Chaos Monkey](http://coreos.com/docs/quickstart/#process-management-with-systemd)
which triggers random reboots, and also does an auto-upgrade to newer releases, which is great for
testing resiliency of my containers.

I had several containers running when CoreOS did an auto-upgrade, and they no longer started after the 
upgrade on Christmas Eve. It took quite a while for me to figure out why they weren't starting, and how 
to fix 

# tl;dr

See `sed` commands at bottom to clean up the container metadata.  Be sure to restart the docker server
to ensure the metadata is re-read before trying to restart the containers.

# Detailed troubleshooting

Here's the problem I saw:

	core@localhost ~ $ docker -v
	Docker version 0.7.2, build 28b162e
	core@localhost ~ $ docker start mysql_latest
	mysql_latest
	core@localhost ~ $ docker ps
	CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
	core@localhost ~ $ docker logs mysql_latest 
	…
	flag provided but not defined: -i
	Usage of /.dockerinit:
	  -g="": gateway address
	  -u="": username or uid
	  -w="": workdir
  
Following suggestions on the mailing list, I ran `docker inspect` and looked at the `Dockerfile`:
  
it looks like the running container is using the 0.7.1 dockerinit, but 0.7.2 is calling that with the new -i option which 0.7.1 doesn't understand.
  
	$ docker inspect mysql_latest
	...
	"SysInitPath": "/var/lib/docker/init/dockerinit-0.7.1",

There are two dockerinit versions available:

	$ sudo ls -al /var/lib/docker/init
	total 9660
	drwx------ 2 root root    4096 Dec 27 23:24 .
	drwx------ 8 root root    4096 Dec 28 21:34 ..
	-rwx------ 1 root root 3941632 Dec 15 05:35 dockerinit-0.7.1
	-rwx------ 1 root root 5935167 Dec 27 23:24 dockerinit-0.7.2

And the calling parameters have definitely changed:

	$ sudo /var/lib/docker/init/dockerinit-0.7.1 -h
	Usage of /var/lib/docker/init/dockerinit-0.7.1:
	  -g="": gateway address
	  -u="": username or uid
	  -w="": workdir
	$ sudo /var/lib/docker/init/dockerinit-0.7.2 -h
	Usage of /var/lib/docker/init/dockerinit-0.7.2:
	  -g="": gateway address
	  -i="": ip address
	  -privileged=false: privileged mode

I can confirm that the CoreOS version changed when this happened:

Check the version in `/etc/os-release`

	 cat /etc/os-release
	NAME=CoreOS
	ID=coreos
	VERSION=176.0.0
	VERSION_ID=176.0.0
	BUILD_ID=176.0.0
	PRETTY_NAME="CoreOS 176.0.0 (Official Build) dev-channel amd64-generic test"
	ANSI_COLOR="1;32"
	HOME_URL="http://www.coreos.com/

Need to mount the old root partition to get the old os version

	$ mount | grep sda[34]
	/dev/sda4 on / type ext4 (ro,relatime)

Looks like we're currently running on sda4, so the previous root partition must be sda3 
(see [Recoverable System Upgrades · CoreOS](http://coreos.com/blog/recoverable-system-upgrades/) for details on the partition layout)

	$ sudo mount -o ro /dev/sda3 /tmp/mnt
	core@localhost ~/docker-rjhsite/mysql $ cat /tmp/mnt/etc/os-release
	NAME=CoreOS
	ID=coreos
	VERSION=160.0.1
	VERSION_ID=160.0.1
	BUILD_ID=160.0.1
	PRETTY_NAME="CoreOS 160.0.1 (Official Build) dev-channel amd64-generic test"
	ANSI_COLOR="1;32"
	HOME_URL="http://www.coreos.com/"

I tried to figure out how to update the container metadata:

	$ CID=`docker ps -a -notrunc | awk '/mysql_latest/ {print $1}'`
	187d09ca14889504f97f7585df0a2ce507bdfce353ce556b07d62cba099e2628
	$ sudo ls -al /var/lib/docker/containers/$CID
	total 60
	drwx------   2 root root  4096 Dec 29 10:48 .
	drwx------ 127 root root 20480 Dec 29 10:43 ..
	-rw-------   1 root root  6985 Dec 29 10:49 187d09ca14889504f97f7585df0a2ce507bdfce353ce556b07d62cba099e2628-json.log
	-rw-------   1 root root   179 Dec 29 10:49 config.env
	-rw-r--r--   1 root root  1693 Dec 29 10:49 config.json
	-rw-r--r--   1 root root  4402 Dec 29 10:49 config.lxc
	-rw-r--r--   1 root root   190 Dec 29 10:49 hostconfig.json
	-rw-r--r--   1 root root    13 Dec 29 10:49 hostname
	-rw-r--r--   1 root root   180 Dec 29 10:49 hosts
	
	core@localhost ~ $ sudo grep --recursive --files-with-matches dockerinit /var/lib/docker/containers/$CID
	/var/lib/docker/containers/$CID/config.lxc
	/var/lib/docker/containers/$CID/$CID-json.log
	/var/lib/docker/containers/$CID/config.json

And `config.lxc` and `config.json` both had references to docker-0.7.1, but when I updated them, and tried `docker start` they reverted immediately to the dockerinit-0.7.1 version.

	/var/lib/docker/containers/$CID/config.lxc:# Inject dockerinit
	/var/lib/docker/containers/$CID/config.lxc:lxc.mount.entry = /var/lib/docker/init/dockerinit-0.7.1 /var/lib/docker/aufs/mnt/$CID/.dockerinit none bind,ro 0 0
	/var/lib/docker/containers/$CID/$CID-json.log:{"log":"Usage of /.dockerinit:\n","stream":"stderr","time":"2013-12-27T23:32:48.761463687Z"}
	/var/lib/docker/containers/$CID/$CID-json.log:{"log":"Usage of /.dockerinit:\n","stream":"stderr","time":"2013-12-27T23:33:58.496630761Z"}
	/var/lib/docker/containers/$CID/$CID-json.log:{"log":"Usage of /.dockerinit:\n","stream":"stderr","time":"2013-12-28T00:30:12.460104457Z"}
	/var/lib/docker/containers/$CID/$CID-json.log:{"log":"Usage of /.dockerinit:\n","stream":"stderr","time":"2013-12-28T21:38:14.877067283Z"}
	/var/lib/docker/containers/$CID/$CID-json.log:{"log":"Usage of /.dockerinit:\n","stream":"stderr","time":"2013-12-28T22:37:48.042034488Z"}
	/var/lib/docker/containers/$CID/$CID-json.log:{"log":"Usage of /.dockerinit:\n","stream":"stderr","time":"2013-12-29T10:49:32.77243386Z"}
	/var/lib/docker/containers/$CID/config.json:{...,"SysInitPath":"/var/lib/docker/init/dockerinit-0.7.1",...}

I poked around in the [dotcloud/docker](https://github.com/dotcloud/docker) source code on GitHub, and 
found various references to `dockerinit`, `config.lxc` and `config.json`, especially within `container.go`
but no smoking gun making it obvious what was happening.

I tried with a simpler container:

	$ CID=`docker ps -a -notrunc | awk '/condescending_torvalds/ {print $1}'`
	$ sudo sed -i 's#/var/lib/docker/init/dockerinit-0.7.1#/var/lib/docker/init/dockerinit-0.7.2#' /var/lib/docker/containers/$CID/config.lxc
	$ sudo sed -i 's#/var/lib/docker/init/dockerinit-0.7.1#/var/lib/docker/init/dockerinit-0.7.2#' /var/lib/docker/containers/$CID/config.json
    
	$ sudo ls -al /var/lib/docker/containers/$CID
	total 52
	drwx------   2 root root  4096 Dec 29 12:17 .
	drwx------ 127 root root 20480 Dec 29 10:43 ..
	-rw-------   1 root root  1952 Dec 29 12:16 767a7c290448392bd519cb3197ee36896bd913594d94ffe1131a7901b4d62939-json.log
	-rw-------   1 root root   192 Dec 29 12:16 config.env
	-rw-r--r--   1 root root  1353 Dec 29 12:17 config.json
	-rw-r--r--   1 root root  3774 Dec 29 12:17 config.lxc
	-rw-r--r--   1 root root   122 Dec 29 12:16 hostconfig.json
	-rw-r--r--   1 root root    13 Dec 29 12:16 hostname
	-rw-r--r--   1 root root   181 Dec 29 12:16 hosts

As soon as the container started, the files were rewritten, and they're back to `dockerinit-0.7.1`:

	core@localhost ~/docker $ docker start condescending_torvalds
	condescending_torvalds
	core@localhost ~/docker $ sudo ls -al /var/lib/docker/containers/$CID
	total 52
	drwx------   2 root root  4096 Dec 29 12:17 .
	drwx------ 127 root root 20480 Dec 29 10:43 ..
	-rw-------   1 root root  2442 Dec 29 12:20 767a7c290448392bd519cb3197ee36896bd913594d94ffe1131a7901b4d62939-json.log
	-rw-------   1 root root   192 Dec 29 12:20 config.env
	-rw-r--r--   1 root root  1352 Dec 29 12:20 config.json
	-rw-r--r--   1 root root  3774 Dec 29 12:20 config.lxc
	-rw-r--r--   1 root root   122 Dec 29 12:20 hostconfig.json
	-rw-r--r--   1 root root    13 Dec 29 12:20 hostname
	-rw-r--r--   1 root root   181 Dec 29 12:20 hosts

I wondered if the docker server is caching this information in memory?  Try making the dockerinit executable
change in `config.*` and restart the docker server before trying to start a container?

	sudo killall docker
	$ sudo sed -i 's#/var/lib/docker/init/dockerinit-0.7.1#/var/lib/docker/init/dockerinit-0.7.2#' /var/lib/docker/containers/$CID/config.lxc
	$ sudo sed -i 's#/var/lib/docker/init/dockerinit-0.7.1#/var/lib/docker/init/dockerinit-0.7.2#' /var/lib/docker/containers/$CID/config.json
	$ sudo nohup /usr/bin/docker -d -r=false &
	$ docker start $CID

**Bingo!**

# Solution

	sudo grep -rl dockerinit-0.7.1 /var/lib/docker/containers | xargs -n 1 sudo sed -i.bak 's/dockerinit-0.7.1/dockerinit-0.7.2/'

Restart the docker server and the containers now start.

	$ docker start mysql_latest edtech_latest proxy_latest
	mysql_latest
	edtech_latest
	proxy_latest
	core@localhost ~ $ docker ps
	CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS              PORTS                      NAMES
	dca29265e76c        nginx-proxy:latest       /bin/sh -c /usr/bin/   4 days ago          Up 3 seconds        0.0.0.0:80->80/tcp         proxy_latest
	5e4183fe09b9        mediawiki-nginx:latest   /usr/bin/mediawiki-s   4 days ago          Up 3 seconds        172.17.42.1:8080->80/tcp   edtech_latest,proxy_latest/edtech
	187d09ca1488        mysql:latest             /run.sh                4 days ago          Up 3 seconds                                   edtech_latest/mysql,mysql_latest,proxy_latest/edtech/mysql

