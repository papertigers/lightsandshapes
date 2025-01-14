---
title: "Tailscale on OmniOS"
date: 2023-08-04T19:14:45-04:00
category: "tutorial"
tags: ["tailscale", "illumos", "omnios"]
---

![tailscale + omnios](/images/tailscale-omnios.png)

>Tailscale: Secure remote access to shared resources

### Background

It turns out that I have a lot of technical content that lives in various gists,
note taking apps, and local markdown files. There are a variety of technical
how-tos, debugging sessions, and subject matter deep dives that I would like to
start posting here. Recently I have redeployed my off-site backup server at my
friends house. One of the crucial pieces of technology that I am leveraging in
that project is [tailscale][], so I figured it would make a nice first post. I
am going to focus primarily on setting up tailscale in an [OmniOS][] zone.

### Prerequisites

#### zadm
There are many ways to manage zones on an illumos system such as [zonecfg(8)][]
and [zoneadm(8)][], however in this post I am going to be making use of
[zadm(1)][]. If you don't have `zadm` on your system, you can easily install it:

{{< terminal >}}
```
# pkg install zadm
```

By default `zadm` uses *json* for its input/output format. I personally perfer
using [toml][] these days. Since the examples in this post will be shown in
toml, I am going to take a quick detour and show you how to change your
defaults. We simply need to modify the following file to look something like
this:

{{< filename file="/etc/opt/ooce/zadm/zadm.conf" >}}
```json
{
    "CONFIG"  : {
        "format"          : "toml"
    },
    "CONSOLE"  : {
        "auto_connect"    : "off",
        "auto_disconnect" : "on",
        "escape_char"     : "_"
    },
    "SNAPSHOT" : {
        "prefix"          : "zadm__"
    }
}
```

The important part here is that you have `"CONFIG": { "format": "toml" }`
present in the config file. Another great setting here is the ability to
override the `"escape_char"`. Typically this defaults to `~` which can be
annoying if you are trying to break out of a console session and you are
currently logged in over ssh.

#### tun driver

Typically when dealing with VPN like software you create virtual network
devices. These are devices which are not physical network adapters themselves,
but rather they are kernel virtual devices which may sit on top of physical
hardware. The device we are interested in is called a *TUN/TAP* device. This
is a device that allows packets to be delivered to a connected user space
program. A *TUN* device sits at [layer 3][] and allows a user space program
like tailscale to encrypt/decrypt packets on the wire by creating a link between
hosts. To install the driver on OmniOS you can run:

{{< terminal >}}
```
# pkg install pkg:/driver/tuntap
```

### Zone setup

Tailscale can be set up to run directly in the globalzone or in a non
globalzone. I am opting to show you how to deploy it in a zone in this case.

First we are going to set up a *vnic* for the zone from the gz. This is
important if you desire to make your zone an [exitnode][]. If you create the
vnic on demand in the zones configuration and assign it an IP address, then the
`allowed-ips` link-prop will be set. This will prevent you from allowing your
zone to act as a packet forwarder preforming NAT for your *tailnet*. From the
globalzone create a *vnic* over the interface of your choice like so:


{{< note >}}
This blog post originally used `tailscale0` and has been updated to use
`tailnode0`. Using `tailscale0` will cause conflicts with tailscale itself!
{{< /note >}}

{{< terminal >}}
```
# dladm create-vnic -l igb1 tailnode0
```

With the *vnic* created we can move onto creating the zone. This can be
accomplished by running:

{{< terminal >}}
```
# zadm create -b sparse tailnode
```

Note if the above command complained about not having the band installed you can
use `pkg` to install the `pkg:/system/zones/brand/sparse` package.

The command will open your *$EDITOR* to a configuration we can modify. Make it
look something like this swapping settings that make sense for your environment
like `resolvers` and `dns-domain`. I highlighted the important part of the
configuration giving the zone access to `/dev/tun`, as well as assigning it the
*vnic* we created above:

{{< highlight toml "hl_lines=17-21">}}
autoboot="true"
bootargs=""
brand="sparse"
dns-domain="hallownest.zone"
fs-allowed=""
hostid=""
ip-type="exclusive"
limitpriv="default"
pool=""
scheduling-class=""
zonename="tailnode"
zonepath="/zones/tailnode"
resolvers=[
  "10.0.1.2",
]

[[device]]
match="/dev/tun"

[[net]]
physical="tailnode0"
{{< /highlight >}}

Next we can boot the zone and wait for all services to come online:

{{< terminal >}}
```
# zadm boot tailnode
# zlogin tailnode
root@tailnode:~# svcs # wait for everything to transition to online
```

The first thing we should do in our new zone is set up an IP address on the
network and assign a default route:

{{< terminal >}}
```
root@tailnode:~# ipadm create-addr -T static -a 10.0.1.58/24 tailnode0/v4
root@tailnode:~# route -p add default 10.0.1.1
```

### tailscale installation

We could install tailscale from `pkg`, however that version lags behind the
currently available version. Unfortunately there are no **official** builds of
tailscale for illumos. However, [Nahum Shalman][] has been working on  pull
requests to various encompassing projects that will one day get us official
support. Until then we can use the builds from his
[github releases](https://github.com/nshalman/tailscale/releases).

{{< terminal >}}
```
root@tailnode:~# cd /var/tmp/
root@tailnode:/var/tmp# wget -q https://github.com/nshalman/tailscale/releases/download/v1.46.1-sunos/tailscaled-illumos
root@tailnode:/var/tmp# mv tailscaled-illumos tailscaled
root@tailnode:/var/tmp# cp tailscaled tailscale
root@tailnode:/var/tmp# chmod +x tailscale*
root@tailnode:/var/tmp# cp ./tailscale{,d} /opt/tailscale/sbin/
root@tailnode:/var/tmp# mkdir -p /var/lib/tailscale
```

It's worth noting that the release file we downloaded is copied to `tailscale`
and `tailscaled`. The binary acts differently depending on how it is `exec`'d.
I suppose you could make one of these a symlink if you really wanted to. I was
lazy so now the binary exists twice.. whoops!

Next we need to create an SMF manifest for tailscale so that we can have the
service start automatically at boot. Edit the following file:

{{< filename file="/var/tmp/tailscale.xml" >}}
```xml
<?xml version='1.0'?>
<!DOCTYPE service_bundle SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type='manifest' name='export'>
  <service name='ooce/network/tailscale' type='service' version='0'>
    <create_default_instance enabled='true'/>
    <single_instance/>
    <dependency name='network' grouping='require_all' restart_on='error' type='service'>
      <service_fmri value='svc:/milestone/network:default'/>
    </dependency>
    <dependency name='filesystem' grouping='require_all' restart_on='error' type='service'>
      <service_fmri value='svc:/system/filesystem/local'/>
    </dependency>
    <method_context>
      <method_credential group='root' user='root'/>
    </method_context>
    <exec_method name='start' type='method' exec='/opt/tailscale/sbin/tailscaled' timeout_seconds='60'/>
    <exec_method name='stop' type='method' exec=':kill' timeout_seconds='60'/>
    <property_group name='application' type='application'/>
    <property_group name='startd' type='framework'>
      <propval name='duration' type='astring' value='child'/>
      <propval name='ignore_error' type='astring' value='core,signal'/>
    </property_group>
    <stability value='Evolving'/>
    <template>
      <common_name>
        <loctext xml:lang='C'>Tailscale</loctext>
      </common_name>
    </template>
  </service>
</service_bundle>
```

Now we can import this manifest into SMF and ensure it's online:

{{< terminal >}}
```
root@tailnode:/var/tmp# svccfg import tailscale.xml
root@tailnode:/var/tmp# svcs tailscale
STATE          STIME    FMRI
online          7:23:09 svc:/ooce/network/tailscale:default
````

Before we can do anything useful we need to connect to tailscale and login,
which can be done like so:

{{< terminal >}}
```
root@tailnode:~# /opt/tailscale/sbin/tailscale up
```

Congratulations!! You are now running a tailscale node. Connect some more zones,
physical servers, or mobile devices to your network and marvel at how they can
communicate as if they were connected to the same physical network.

You can stop here or you can continue on to the next section if you are
interested in having your zone act as an *exitnode*.


### Running an exitnode

In order for our tailscale node to make a useful exit node we need to set the
interface to preform NAT for us. Luckily this is pretty easy to do on OmniOS.
First we need to tell `ipnat(8)` that we want to map things from the carrier
grade network arriving on the `tailnode0` vnic to anywhere:

{{< filename file="/etc/ipf/ipnat.conf" >}}
```text
map tailnode0 100.64.0.0/10 -> 0/32
```

Then we can enable `ipfilter` (which controls ipnat) and enable forwarding like
so:

{{< terminal >}}
```
root@tailnode:~# svcadm enable ipfilter
root@tailnode:~# svcadm enable ipv4-forwarding
```

Finally we can tell our zone to act as an exitnode (note that you will have to
go approve this configuration in the tailscale web console):

{{< terminal >}}
```
root@tailnode:~# /opt/tailscale/sbin/tailscale set --advertise-exit-node
```

### Conclusion

Tailscale can be really useful software if you want to connect a bunch of
devices over an untrusted network. Like I mentioned, I am using it with my
off-site backup solution to ensure the data being transmitted is encrypted over
the public internet. There are many additional features that tailscale offers
that this post didn't cover. One such feature, `--advertise-routes`, can be
used to provide the devices on your tailnet a route to other resources that are
not directly connected to tailscale themselves. Finally for the extra paranoid
out there if depending on an external service you have zero visibility into is
not your thing, then there is still hope. If this sounds like you, then it may
be worth checking out [headscale][], which is a self-hosted open source
implementation of the tailscale control server.

[tailscale]: https://tailscale.com
[OmniOS]: https://omnios.org
[zonecfg(8)]: https://illumos.org/man/8/zonecfg
[zoneadm(8)]: https://illumos.org/man/8/zoneadm
[zadm(1)]: https://github.com/omniosorg/zadm
[toml]: https://toml.io/
[layer 3]: https://en.wikipedia.org/wiki/Network_layer
[exitnode]: https://tailscale.com/kb/1103/exit-nodes/
[Nahum Shalman]: https://blog.shalman.org
[headscale]: https://headscale.net
