+++
title = "Sanity Checking with the pid Provider"
date = 2013-01-28
category = "debugging"
tags = ["syslog", "illumos", "dtrace", "smartos", "pid provider"]
+++

> I am not crazy right?

It has been a while since I posted an entry, so I thought I would type up a
short post on some fun with the DTrace [pid
Provider](https://illumos.org/books/dtrace/chp-pid.html#chp-pid). I was recently working with
[stud](https://github.com/bumptech/stud) on [SmartOS](http://smartos.org/) and
I wanted to send the logs off to the syslog facility *local4*. So I modified */etc/syslog.conf* with the following.

{{< filename file="/etc/syslog.conf" >}}
```text
local4.* /var/log/stud.log

```

Then I simply ran `svcadm restart system-log` and fired up stud. Running `tail
-f /var/log/stud.log` seemed to reveal that nothing was being logged. My first
thought was that I did not end the file with a new line.  Checking the config
again I made sure that I had the new line at the end. (Older syslog on unix
cares about formatting).

My second thought was lets fire up dtrace and see if the application is
actually logging anything to syslog.  I figured the application was calling a
function in libc to log data out.  To look for the probe name I ran `dtrace -ln
'pid$target::*log*:' -p <pid>` and sure enough there were two interesting
probes to look at, syslog and openlog.  Looking into the syslog:entry probe I
wanted to know what the arguments were so that I could gather some data.  The
online documentation for syslog says _void syslog (int
facility_priority, char \*format, ...)_.  So arg0 will allow me to check the
priority and arg1 will let me see what the message logged is. If there is any
activity at all for that matter.

Checking facility priority:

```
root@server:~#
root@server:~# dtrace -n 'pid$target::syslog:entry {printf("Priority: %d with message: %s", arg0, copyinstr(arg1))}' -p 75576
dtrace: description 'pid$target::syslog:entry ' matched 1 probe
CPU     ID                    FUNCTION:NAME
  6  78675                     syslog:entry Priority: 3 with message: {%s} Connection closed (in data)
```

Great, so according to dtrace stud is indeed producing syslog messages and is
sending them to priority number 3.  According to
[syslog.h](http://src.illumos.org/source/xref/illumos-gate/usr/src/lib/libbc/inc/include/syslog.h)
a priority of 3 is *err*. Double checking my *syslog.conf* file I am looking
for _*_ so I changed it to be _err_.

{{< filename file="/etc/syslog.conf" >}}
```text
local4.err /var/log/stud.log

```

Now that I have the right priority I am still not getting any log data in
*/var/log/stud.log*.

DTrace has verified that logs are being generated and that they are going to
priority 3, but I don't know if the facility is correct.  I still have the
other probe (openlog:entry) to look into. The documentation says **Function:
void openlog (const char \*ident, int option, int facility)**.  This is exactly
what I need, arg2 will tell me what facility the application is actually
attaching itself to. This is a little bit more tricky because it appears the
application will only call this function when its first starting up.  So I
couldn't pass in the pid this time.  Instead I used dtrace's "-c" option to
pass in a command which will grab the pid and stick it in $target. 

Checking the facility:

```
root@server:/var/log#
root@server:/var/log# dtrace -n 'pid$target::openlog:entry {trace(arg2)}' -c "/opt/local/bin/stud --config=/opt/local/etc/stud.conf"
dtrace: description 'pid$target::openlog:entry ' matched 1 probe
prctl: 30062: cannot control: process is traced: Device busy
{bind-socket}: Address already in use
dtrace: pid 30062 has exited
CPU     ID                    FUNCTION:NAME
 11  67108                    openlog:entry                 0
 11  67108                    openlog:entry               160
```

Stud is attaching to facility 160. However
[syslog.h](http://src.illumos.org/source/xref/illumos-gate/usr/src/lib/libbc/inc/include/syslog.h)
in illumos defines facilities as bit shifted int's.  Looking at local4 it is
defined  as (20<<3). I can't really shift bits in my head that easily so a
simple mdb one liner will tell us if the int matches up with int in the
header file.

```
[link@typhoon ~]$ mdb
> 0t20 << 3 =D
                160
```

Success! Stud is indeed using local4 as its facility and it is using priority 3
(err) to send its logs to.  DTrace was able to confirm that the application is
definitely not the issue.  I think having this kind of observability is a clear
win for illumos over other platforms.  Now I know the issue is not stud but has
something to do with my syslog.conf.  Knowing that syslog.conf is picky about
things like new lines at the end of the file, it had to be something to do with
formatting.  Thinking back to other times when I edited syslog.conf I remembered
that syslog likes tab characters and not spaces between the facility.priority
and the log file.  Changing my space to a tab and restarting system-log allowed
the logs to start flooding in.
