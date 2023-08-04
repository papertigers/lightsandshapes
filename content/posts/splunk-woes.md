+++
title = "Splunk Performance on SmartOS"
date = 2012-07-19
category = "tutorial"
tags = ["splunk", "illumos", "dtrace", "smartos"]
+++

>Debugging splunk performance issues in a live environment.

Since joining [Voxer](http://voxer.com) to do performance engineering, I have also become the owner of our splunk instances.  We rely on splunk heavily to monitor our infrastructure just as most companies do.  It is an important tool especially while doing a code push to production.  Splunk gives us observability into how our application is performing without having a million `tail -f /path/to/a.log` windows open. Splunk also makes it easy to correlate the data between the different types of logs we have. 

When you rely so heavily on a tool on a day to day basis it's easy to realize when it is under performing. Splunk, for us, was falling behind on indexing and we didn't know why.  At first glance it appeared that the splunk forwarders were behind in sending their data to the indexers.  Upon closer inspection, the logs on the forwarders showed that the indexers were in a blocked state and the forwarder would move on to a new indexer and try again.  We could have chalked this up to the indexers not being able to keep up with the amount of log data we were generating, and introduced more indexers to load balance across, however that's not a solution. Since we are in an [illumos](http://illumos.org) [environment](http://joyent.com) we have many tools at our disposal to do performance analysis. 

Originally some Splunk Engineers thought that we were disk bound and that the indexers could not keep up.  Looking at `iostat -xzn 1` and `vfsstat 1` I saw the following:    

```
root@somehost:~# iostat -xzn 1
									extended device statistics              
	r/s    w/s   kr/s   kw/s wait actv wsvc_t asvc_t  %w  %b device
	0.1    0.1    0.5    0.2  0.0  0.0    0.0    0.0   0   0 ramdisk1
	4.5   82.1  570.9 6256.1  0.0  0.1    0.0    0.8   0   2 sd1
									extended device statistics              
	r/s    w/s   kr/s   kw/s wait actv wsvc_t asvc_t  %w  %b device
	1.0    4.0  127.3   47.7  0.0  0.0    0.0    2.6   0   1 sd1
									extended device statistics              
	r/s    w/s   kr/s   kw/s wait actv wsvc_t asvc_t  %w  %b device
	0.0    6.0    0.0   32.0  0.0  0.0    0.0    0.0   0   0 sd1
									extended device statistics              
	r/s    w/s   kr/s   kw/s wait actv wsvc_t asvc_t  %w  %b device
	2.0  301.5  255.5 19327.1  0.0  0.1    0.0    0.4   0   4 sd1
									extended device statistics              
	r/s    w/s   kr/s   kw/s wait actv wsvc_t asvc_t  %w  %b device
	1.0    0.0  128.2    0.0  0.0  0.0    0.0   14.0   0   1 sd1
```


The disks were almost always less than ~5% busy so, we were surely not bottlnecked on disk IO. Next I looked at CPU using `prstat -mLc 1` and `mpstat 1` and I didn't see anything that screamed CPU bottelneck. After taking a closer look though I saw splunk maxing out a single core.  Since these boxes have 24 cores, it was easy to over look the fact that splunk was only around ~4% CPU.  When you take 100% and divide it by 24 cores you see that the complete use of one core is ~4.1%. 

I returned to the Splunk Engineers with my findings and we decided to look at where splunk was spending its time.  How do you look at where splunk is spending its time? Well, with splunk of course! A good way to do this is to run the following query `index=_internal source=*metrics.log group=pipeline name=typing NOT sendout | timechart span=1m sum(cpu_seconds) by processor`.  This query will show you where splunk is spending its CPU cycles.  In our case splunk seemed to be busy doing regex work. At this point we made a bunch of changes to different conf files in splunk to remove regex where we could and make regex more efficient in the places we couldn't.

Making these changes was easy in our environment since we are using [chef](http://opscode.com). Once the chef run finished I was able to determine that the changes we made did in fact cut down on regex time according to the query, but data being received from the forwarders was behind sometimes by hours.  What's worse is that some data would load in out of order, making it look like we were missing chunks of data.  This is worse than having no data at all.

At this point I thought I was out of options compeletly, and would have to give in to adding more indexers, even though the indexers did not appear to be bound by any resource.  To double check myself, I reached out to Joyent to make sure iostat was returning the correct information and there wasn't something funky going on with it in a zone. Luckily [Bryan Cantrill](http://dtrace.org/blogs/bmc) got back to me and confirmed that I was not insane and that IO throughput looked fine.  However, the thing he mentioned next was very important.  He noticed that splunk was running with the *UMEM_DEBUG* env variable set.  He explained that this would put libumem in a debug mode that would introduce some overhead in malloc(3C) and free(3C) calls. 

Finally! I was on to something.  Using `pargs` I was able to confirm his findings.  Dumping splunks env variables you can see UMEM_DEBUG is set to "default".

```
root@somehost ~ # pargs -e 5167
5167:   splunkd -p 9079 start
envp[0]: _=*7372*/splunk/splunkforwarder/bin/splunk
envp[1]: LANG=en_US.UTF-8
envp[2]: PATH=/splunk/splunkforwarder/bin:/usr/sbin:/usr/bin
envp[3]: PWD=/splunk
envp[4]: SHLVL=1
envp[5]: SMF_FMRI=svc:/application/splunkforwarder:default
envp[6]: SMF_METHOD=start
envp[7]: SMF_RESTARTER=svc:/system/svc/restarter:default
envp[8]: SMF_ZONENAME=<ZONE UUID>
envp[9]: TZ=UTC
envp[10]: UMEM_DEBUG=default
envp[11]: A__z="*SHLVL
envp[12]: HOSTNAME=somehost.company.com
envp[13]: SPLUNK_HOME=/splunk/splunkforwarder
envp[14]: SPLUNK_DB=/splunk/splunkforwarder/var/lib/splunk
envp[15]: SPLUNK_SERVER_NAME=splunkforwarder
envp[16]: SPLUNK_WEB_NAME=splunkweb
envp[17]: LD_LIBRARY_PATH=/splunk/splunkforwarder/lib
envp[18]: LD_LIBRARY_PATH_64=/splunk/splunkforwarder/lib
envp[19]: LD_PRELOAD=libumem.so
envp[20]: LDAPCONF=/splunk/splunkforwarder/etc/openldap/ldap.conf
```

What does that mean though?  Referencing the the man pages for UMEM_DEBUG(3MALLOC),  "default" sets a few things (audits, contents, and guards). For example audits enables the recording of auditing information, including thread ID, high-resolution time stamp, and stack  trace  for  the  last action  (allocation  or  free) on every allocation. Ahhh, there is some of that overhead Bryan mentioned. Lets take a peek at how long a malloc call by splunk takes so we can compare it to when UMEM_DEBUG is not set.

Before disabling UMEM_DEBUG,  `dtrace -n 'pid$target::malloc:entry {self->ts = timestamp} pid$target::malloc:return /self->ts/{this->delta = timestamp - self->ts; self->ts = 0; @["lat"] = avg(this->delta)} tick-5s {printa(@["lat"]); exit(0)}' -p 12776` produces the following output:

```
dtrace: description 'pid$target::malloc:entry ' matched 7 probes
CPU     ID                    FUNCTION:NAME
	5  55472                         :tick-5s 
	lat                                                            2397
```

We know that UMEM_DEBUG is introducing latency for our mallocs/frees.  We can check how many mallocs splunk is doing a second with some more dtrace.

```
root@somehost:~#
root@somehost:~# dtrace -n 'pid$target::malloc:entry {@malloc = count()} tick-1s {printa(@malloc); exit(0)}' -p 25974
dtrace: description 'pid$target::malloc:entry ' matched 4 probes
CPU     ID                    FUNCTION:NAME
	6  55499                         :tick-1s 
	malloc                                                         1362 
```

That's a lot of mallocs in one second that we could be speeding up by reducing this debug overhead.  I then needed to track down what was setting this env variable.  These values can be set by doing something like `UMEM_DEBUG=default LD_PRELOAD=libumem.so splunk start`.  Since we are not setting this variable anywhere else for any process it must be in the start script for splunk.  So, I turned to SMF for answers.  `svccfg export splunkforwarder` returned the following config:  

{{< filename file="splunkforwarder.xml" >}}
```xml
<?xml version='1.0'?>
<!DOCTYPE service_bundle SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type='manifest' name='export'>
  <service name='application/splunkforwarder' type='service' version='0'>
    <create_default_instance enabled='true'/>
    <single_instance/>
    <dependency name='network' grouping='require_all' restart_on='error' type='service'>
      <service_fmri value='svc:/milestone/network:default'/>
    </dependency>
    <dependency name='filesystem' grouping='require_all' restart_on='error' type='service'>
      <service_fmri value='svc:/system/filesystem/local'/>
    </dependency>
    <method_context working_directory=':default'>
      <method_credential group='splunk' user='splunk'/>
      <method_environment>
        <envvar name='PATH' value='/opt/local/bin:/opt/local/sbin:/usr/bin:/usr/sbin'/>
        <envvar name='HOME' value='/splunk'/>
      </method_environment>
    </method_context>
    <exec_method name='start' type='method' exec='/splunk/splunkforwarder/bin/splunk %m --accept-license' timeout_seconds='300'>
      <method_context>
        <method_environment>
          <envvar name='UMEM_DEBUG' value='default'/>
        </method_environment>
      </method_context>
    </exec_method>
    <exec_method name='stop' type='method' exec='/splunk/splunkforwarder/bin/splunk %m' timeout_seconds='300'/>
    <exec_method name='refresh' type='method' exec='/splunk/splunkforwarder/bin/splunk restart' timeout_seconds='600'/>
    <stability value='Evolving'/>
    <template>
      <common_name>
        <loctext xml:lang='C'>splunkforwarder - Splunk Forwarder</loctext>
      </common_name>
    </template>
  </service>
</service_bundle>
```

Ah ha!  What's the `<envvar name='UMEM_DEBUG' value='default'/>` bit doing in there? I figured it must be carry over for us when we first moved to Joyent and changed our services to use SMF instead of upstart.  I checked with one of our engineers, [dshaw](http://dshaw.com), and he explained to me the process of how splunk got setup. The manifest was based around the one found on [splunkbase.](http://splunk-base.splunk.com/answers/6152/is-there-a-solaris-smf-manifest-for-splunk)  This knowledge base has served 1,830 people at the time of writing this article (UPDATE The splunkbase has since been updated to reflect these findings).  That's potentially a lot of misconfigured splunk instances out there in illumos(solaris) environments.

After changing the SMF Manifest to drop `<method_context> ... <envvar name='UMEM_DEBUG' value='default'/> ... </method_context>`  I pushed out the new configs via chef again. Now back to dtrace to figure out how much of a performance gain we will get and to see if its the BIG win we are hoping for.

After disabling UMEM_DEBUG,  `dtrace -n 'pid$target::malloc:entry {self->ts = timestamp} pid$target::malloc:return /self->ts/{this->delta = timestamp - self->ts; self->ts = 0; @["lat"] = avg(this->delta)} tick-5s {printa(@["lat"]); exit(0)}' -p 12776` produces the following output:    

```
dtrace: description 'pid$target::malloc:entry ' matched 7 probes
CPU     ID                    FUNCTION:NAME
 13  73567                         :tick-5s 
  lat                                                            1596
```

Lets check how many more mallocs we can do a second without UMEM_DEBUG.

```
root@somehost:~#
root@somehost:~# dtrace -n 'pid$target::malloc:entry {@malloc = count()} tick-1s {printa(@malloc); exit(0)}' -p 28177
dtrace: description 'pid$target::malloc:entry ' matched 4 probes
CPU     ID                    FUNCTION:NAME
 17  55499                         :tick-1s 
  malloc                                                        11257
```

We jumped from 1362 mallocs to 11257 mallocs in a second.  This means with UMEM_DEBUG enabled we were operating at 1/7th of our possible throughput.  Talk about performance gain!  The latency of the malloc call dropped from 2397ns to 1596ns. 

Not only do we get a giant performance gain by removing this env variable we will soon get an ever bigger performance boost from [Per-thread caching in libumem.](http://dtrace.org/blogs/rm/2012/07/16/per-thread-caching-in-libumem/)  This was another issue discovered by us when trying to figure out why some of our processes were faster on linux than on smartos.  I hope to blog about some benchmarks before and after our upgrade to the newest smartos platform when joyent makes that available.  
