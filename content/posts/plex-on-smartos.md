+++
title = "Plex on SmartOS"
date = 2015-04-04
category = "tutorial"
tags = ["plex", "illumos", "lx brand", "smartos"]
+++

>Running Plex securely on top of ZFS through the magic of lx branded zones and SmartOS. 

The nice folks at [Plex](https://plex.tv/features) recently released a version
of [Plex Media Server](https://plex.tv/downloads) for SmartOS and other illumos
variants.  Actually, half of that statement is a lie.  While the folks over at
Plex are nice, they didn't actually release a version of Plex for the illumos
family.  Some of you may be thinking  whats the point of this blog post?  Well
I am here to tell you that Plex does indeed run on top of SmartOS. "How?", you
may ask.  The answer to that is *lx branded zones*.

Normally when you provision a zone on SmartOS you specify `"brand": "joyent"`
or `"brand": "kvm"` in your create payload. What this really tells `vmadm(1M)`,
is to create a zone using the joyent|kvm *brand*.  Brands are described by
**brand(5)** as "alternate operating environments for non-global zones". The
man page goes on to further explain what this means by stating "In addition, a
zone's brand is used to properly identify the correct application type at
application launch time."  Long story short, the *brand* is used to setup the
zone with a particular environment in mind. Currently in smartos this value may
be one of 'joyent', 'joyent-minimal', 'lx',  or 'sngl' for OS virtualization
and 'kvm' for full hardware virtualization.

For the purpose of this post we are going to concentrate on the 'lx' brand.  An
lx branded zone is a zone that provides a container in which the user is
presented with a linux environment. The lx brand allows you to run Linux binary
applications unmodified without sacrificing the IO performance you generally
lose inside of hardware virtualization.  It's important to note that the linux
kernel itself is missing in this environment. This means you get all the
benefits of the SmartOS kernel along with key technologies such as ZFS, DTrace,
mdb, and crossbow coupled with your linux binaries. [Joyent](http://joyent.com)
has spent a lot of time reviving the lx brand work from  the old Sun
Mircosystems days and bringing it into the modern era with support for things
like 64bit, epoll, and inotify. You can find out much more about the lineage of
the technology from [Bryan Cantrill's](http://dtrace.org/blogs/bmc/) talk
[here](https://www.youtube.com/watch?v=TrfD3pC0VSs).

Enough background information, lets move on to the fun stuff, aka setting up
Plex on SmartOS.  For the purpose of this post I will assume you have SmartOS
running already or you know how to use it already.  The first thing you need to
do is make sure that you are on a relatively new platform, the newer the better
because new fixes are going into the lx brand work all the time.  Secondly you
want to import an lx image of your choice.  For my purposes I generally prefer
ubuntu or debian so that's what I will demonstrate here.

```
imgadm sources -a https://updates.joyent.com
imgadm import 818cc79e-ceb3-11e4-99ee-7bc8c674e754 # lx-ubuntu-14.04 20150320
imgadm import 116deb8c-cf03-11e4-9b2d-7b1066800a6a # lx-debian-7 20150320
```

Next you will need to create a file named `plex.json` with the following contents.

{{< filename file="plex.json" >}}
```json
{
  "alias": "plex",
  "brand": "lx",
  "kernel_version": "3.13.0",
  "max_physical_memory": 4096,
  "image_uuid": "818cc79e-ceb3-11e4-99ee-7bc8c674e754",
  "resolvers": ["8.8.8.8","8.8.4.4"],
  "nics": [
    {
      "nic_tag": "admin",
      "ip": "192.168.128.25",
      "netmask": "255.255.255.0",
      "gateway": "192.168.128.1",
      "primary": "1"
    }
  ],
  "filesystems": [
    {
      "type": "lofs",
      "source": "tank/Movies",
      "target": "/Movies"
    },
        {
      "type": "lofs",
      "source": "tank/TV",
      "target": "/TV"
    },
        {
      "type": "lofs",
      "source": "tank/Music",
      "target": "/Music"
    }
  ]
}
```

Lets discuss quickly the *filesystems* section of the json file.  Basically
what this is telling smartos to do is mount in from the zpool named tank some
ZFS filesystems and under what path they should be mounted locally to the zone.
Since we are in an OS container rather than a hardware virtualized VM our zone
will have direct access to the ZFS datasets.  In the json file we could also
pass in some mounting options such as 'ro' if we wanted to lock down what the
zone could do.  It's also entirely possible to give the zone delegated access
over those filesystems allowing the zone to set properties such as compression
or take snapshots.  So, you will want to modify the *filesystems* section to
point to any ZFS filesystems you might already have with media in them.  You
will also want to modify the *nics* section of the json file to match your
networking needs.  For example, my Plex zone at home is given a nic on a dmz
network.

Our next step is to actually create the zone.

```
vmadm validate create -f plex.json
vmadm create -f plex.json
	....let the zone provision....
vmadm list | grep plex
```

Now that we have our `uuid` for the zone we can go ahead and login at this
point to poke around and install Plex.

```
[smartos ~]# vmadm list | grep plex
51d28bb3-7b5c-cefe-d7b9-b9683085856e  LX    4096     running           plex
[smartos ~]# zlogin 51d28bb3-7b5c-cefe-d7b9-b9683085856e
[Connected to zone '51d28bb3-7b5c-cefe-d7b9-b9683085856e' pts/4]
Last login: Sat Apr  4 22:17:36 UTC 2015 from zone:global on pts/14
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0 x86_64)

 * Documentation:  https://help.ubuntu.com/
   __        .                   .
 _|  |_      | .-. .  . .-. :--. |-
|_    _|     ;|   ||  |(.-' |  | |
  |__|   `--'  `-' `;-| `-' '  ' `-'
				   /  ;  Instance (Ubuntu 14.04 LX Brand 20150320)
				   `-'   https://docs.joyent.com/images

root@plexlx:~#
root@plexlx:~# uname -a
Linux plexlx 3.13.0 BrandZ virtual linux x86_64 x86_64 x86_64 GNU/Linux
root@plexlx:~# cat /etc/issue
Ubuntu 14.04.2 LTS \n \l
root@plexlx:~# ls /
bin   dev  home  lib64  mnt     Music   opt   root  sbin  sys     tmp  usr
boot  etc  lib   media  Movies  native  proc  run   srv   system  TV   var
```

Awesome! We now have a zone running Ubuntu 14.04 with three ZFS filesystems
mounted inside of it. Next we have to install the plex media server package.

```
root@plexlx:~# wget https://downloads.plex.tv/plex-media-server/0.9.11.16.958-80f1748/plexmediaserver_0.9.11.16.958-80f1748_amd64.deb
--2015-04-05 04:52:29--  https://downloads.plex.tv/plex-media-server/0.9.11.16.958-80f1748/plexmediaserver_0.9.11.16.958-80f1748_amd64.deb
Resolving downloads.plex.tv (downloads.plex.tv)... 2400:cb00:2048:1::6814:709, 2400:cb00:2048:1::6814:609, 104.20.7.9, ...
Connecting to downloads.plex.tv (downloads.plex.tv)|2400:cb00:2048:1::6814:709|:443... failed: Network is unreachable.
Connecting to downloads.plex.tv (downloads.plex.tv)|2400:cb00:2048:1::6814:609|:443... failed: Network is unreachable.
Connecting to downloads.plex.tv (downloads.plex.tv)|104.20.7.9|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 119172228 (114M) [application/octet-stream]
Saving to: ‘plexmediaserver_0.9.11.16.958-80f1748_amd64.deb’

100%[==============================================================================================================>] 119,172,228 21.8MB/s   in 12s

2015-04-05 04:52:41 (9.45 MB/s) - ‘plexmediaserver_0.9.11.16.958-80f1748_amd64.deb’ saved [119172228/119172228]

root@plexlx:~# dpkg -i plexmediaserver_0.9.11.16.958-80f1748_amd64.deb
Selecting previously unselected package plexmediaserver.
(Reading database ... 18097 files and directories currently installed.)
Preparing to unpack plexmediaserver_0.9.11.16.958-80f1748_amd64.deb ...
Unpacking plexmediaserver (0.9.11.16.958-80f1748) ...
Setting up plexmediaserver (0.9.11.16.958-80f1748) ...
OK
plexmediaserver start/running, process 91040
Processing triggers for ureadahead (0.100.0-16) ...
Processing triggers for mime-support (3.54ubuntu1) ...
root@plexlx:~# service plexmediaserver status
plexmediaserver start/running, process 91040
```

Finally all you have to do is setup the server.  The way that I usually do this
is through a ssh tunnel.  Which looks something like `ssh -L
8000:localhost:34200 root@192.168.128.25`.  From here you can point your
browser to [http://localhost:8000/web](http://localhost:8000/web).  At this
point if you do not have a plex account you will need to make one.  Then all
that's left to do is add the media through the webui and start enjoying some of
your content.

### Bonus

Above I mentioned that lx branded zones allowed you to use great tools like
DTrace to debug linux applications.  Without taking too much of a deep dive
into DTrace or plex internals I thought I would just show some basics.  The lx
branded zones have a special path mounted into their filesystem called
`/native` this is where all the native smartos stuff lives.  Running different
binaries and tools out of this path is not recommended.  However some tools
such as dtrace, prstat, pfiles, and pargs are known to work very well.  A
convenient way to access this is by sticking the tools at the end of your
$PATH by adding `/native/usr/bin`, `/native/usr/sbin`, and `/native/sbin`.

```
root@plexlx:~# pgrep -l -f Plex
5473 Plex Media Serv
40425 python
root@plexlx:~# dtrace -n 'lx-syscall:::entry /pid == $1/ {@[probefunc] = count()} tick-10sec {exit(0)}' 5473
dtrace: description 'lx-syscall:::entry ' matched 677 probes
CPU     ID                    FUNCTION:NAME
  1  79593                      :tick-10sec

  munmap                                                            3
  exit                                                              5
  open                                                             16
  kill                                                             31
  nanosleep                                                        31
  accept                                                           39
  ioctl                                                            39
  shutdown                                                         39
  openat                                                           67
  getpeername                                                      78
  getsockname                                                      78
  recvmsg                                                         100
  clock_gettime                                                   117
  fcntl                                                           121
  close                                                           126
  getdents                                                        134
  read                                                            152
  madvise                                                         170
  epoll_ctl                                                       197
  write                                                           249
  epoll_wait                                                      263
  sendmsg                                                         274
  futex                                                           828
  stat                                                           2464

root@plexlx:~# dtrace -n 'lx-syscall::read:entry /pid == $1/ {trace(fds[arg0].fi_pathname)}' 5473
dtrace: description 'lx-syscall::read:entry ' matched 2 probes
CPU     ID                    FUNCTION:NAME
  2   4654                       read:entry   /var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Plug-in Support/Databases/com.plexapp.plugins.library.db
  6   4654                       read:entry   /var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Plug-in Support/Databases/com.plexapp.plugins.library.db
  3   4654                       read:entry   /run/resolvconf/resolv.conf_new.5364
  3   4654                       read:entry   /run/resolvconf/resolv.conf_new.5364
  3   4654                       read:entry   /etc/hosts
  3   4654                       read:entry   /etc/hosts
  ............(cut output).........
 12   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
 12   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
  4   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
  4   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
 16   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
 16   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
 16   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts
 17   4654                       read:entry   /tmp/plex-transcode-557A1286-636B-4FAA-AB07-385AF8CBE625-00-f044e357-ddf2-4107-865c-6ed473b9cc06/media-00005.ts




root@plexlx:~# pargs -ae 5473
5473:   ./Plex Media Server
argv[0]: ./Plex Media Server

envp[0]: UPSTART_INSTANCE=
envp[1]: LD_LIBRARY_PATH=/usr/lib/plexmediaserver
envp[2]: HOME=/var/lib/plexmediaserver
envp[3]: PLEX_MEDIA_SERVER_TMPDIR=/tmp
envp[4]: OLDPWD=/
envp[5]: UPSTART_JOB=plexmediaserver
envp[6]: TMPDIR=/tmp
envp[7]: PATH=/usr/local/sbin:/usr/local/bin:/usr/bin:/usr/sbin:/sbin:/bin
envp[8]: PLEX_MEDIA_SERVER_APPLICATION_SUPPORT_DIR=/var/lib/plexmediaserver/Library/Application Support
envp[9]: PLEX_MEDIA_SERVER_MAX_STACK_SIZE=3000
envp[10]: IFACE=eth0
envp[11]: UPSTART_EVENTS=filesystem net-device-up
envp[12]: PWD=/usr/lib/plexmediaserver
envp[13]: PLEX_MEDIA_SERVER_HOME=/usr/lib/plexmediaserver
envp[14]: PLEX_MEDIA_SERVER_MAX_PLUGIN_PROCS=6
```

### Stability
Plex has been relatively stable for me on recent releases of SmartOS.  I have
had one core file generated since standing up my zone.  Other than that one
crash plex has been stable for me with multiple people streaming content at
once.  That's not to say that early on we didn't have issues.  There has been
some key commits to making Plex stable on SmartOS such as this recent fix on
[github](https://github.com/joyent/illumos-joyent/commit/a9a452609b1832657d68c06f674196b4db54a5d6).
The final gotcha seems to be that DLNA support is not working.


### Closing Thoughts

If you are interested in any of the lx work Joyent is doing you can continue to
monitor the [github
page](https://github.com/joyent/illumos-joyent/commits/master).  Its worth
mentioning that Joyent started doing this lx brand work in order to make
[Docker](https://www.docker.com/) a reality on top of
[SDC](https://github.com/joyent/sdc) as well as the supported offering called
[Triton](https://www.joyent.com/about/press/joyent-launches-triton-elastic-container-infrastructure)
in the Joyent Public Cloud.  Expect more blog entries from me in the near
future on using Docker with lx branded zones and Triton.

Finally I want to say thank you to the supportive Plex developers on irc.  If
there is interest in a native build of Plex for the illumos family(SmartOS,
OmniOS, OpenIndiana etc) please voice your opinion here or on twitter.  It
would be great to have a native build, but until then live the dream and run
Plex on lx branded zones!
