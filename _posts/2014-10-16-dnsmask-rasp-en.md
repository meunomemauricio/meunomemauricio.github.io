---
layout: post
comments: true
title: Configurando Servidor DNS e DHCP no Raspberry Pi (dnsmasq)
date: 2014-10-16 12:00:00
lang: en
ref: dnsmasq-rasp
categories: Linux
tags:
- Raspberry Pi
- Networking
- Linux
excerpt_separator: <!--more-->
---

The [DNSMasq][dnsmasq] is a service that combines DNS and DHCP in an elegant
and easy to configure way. It's focus is to consume as little memory resources
and disk space as possible, therefore it's highly recommended for simple
networks with only a couple of hosts.

With those characteristics, it's very common to see it being used on Linux
distributions aimed at home routers, like the [OpenWrt][openwrt] or
[DD-WRT][ddwrt]. For the same reason, it's also perfect to be run on a
[Raspberry Pi][rasppi].

In this post I'll run through and explain a simple configuration that I have
used for my local network.

<!--more-->

The DNSMasq is available as a `.deb` package for the [Raspbian][raspbian], so
it can be installed through an apt-get:

{% highlight terminal %}
$ sudo apt-get install dnsmasq
{% endhighlight %}

The main configuration file is on `/etc/dnsmasq.conf`. It's possible to write
the whole configuration directly in it, nonetheless, it's better to create new
files in the `/etc/dnsmasq.d` directory for organization. In order for this to
be possible, it's necessary to uncomment the last line of the
`/etc/dnsmas.conf` file:

{% highlight ini %}
# Include a another lot of configuration options.
#conf-file=/etc/dnsmasq.more.conf
conf-dir=/etc/dnsmasq.d
{% endhighlight %}

From now on, every text file created on `/etc/dnsmasq.d` will be used as the
service configuration:

{% highlight terminal %}
$ sudo vi /etc/dnsmasq.d/network.conf
{% endhighlight %}

> I have used `network.conf` for the filename, but anything else is fine.

The first step is to configure a **domain name**. It's defined in the `domain`
option:

{% highlight ini %}
domain=raspberry.local
{% endhighlight %}

This domain dame will be accessible only locally and through this DNS server.

By default, the DNSMasq will use the `nameserver` addresses defined in the
`/etc/resolve.conf` as the outside DNS Servers. But there is also the
`no-resolv` option that disables that behaviour and then it's possible to
configure the DNS manually:

{% highlight ini %}
no-resolv
server=208.67.222.222
server=208.67.220.220
{% endhighlight %}

I have used the [OpenDNS][opendns] instead of the DNS provided by my ISP. 
[GoogleDNS][googledns] is also another good option.

> For more configurations on the DNS service, [consult the DNSMasq
> manual page][manual].

Following the DNS configuration, we follow on to the DHCP configuration. The
first configuration is the `dhcp-range` that defines a range of IP addresses
that are going to be distributed among the clients when joining the network:

{% highlight ini %}
dhcp-range=10.0.0.50,10.0.0.59,120m
{% endhighlight %}

In this case, the server will assign addresses from `10.0.0.50` up to
`10.0.0.59`, inclusive. It's important to adjust the range according to how
many devices will be connected to the network simultaneously.

It's also important to notice the *120m* value configured together with the
address range. This value is the **lease time** for this range.

> The **lease time** configuration is something that varies with application.
> The most usual value is 24h, which is already pretty adequate for most home
> networks. Shorter periods are usually used for public WiFi networks, like in
> stores and restaurants, where there is a great rotation of clients accessing
> the network.

After defining the DHCP range and the lease time, we can define the default
gateway that will be advertised to the clients:

{% highlight ini %}
dhcp-option=option:router,10.0.0.254
dhcp-authoritative
{% endhighlight %}

There is no direct configuration for the default gateway but it's possible to
use the `dhcp-option`, which directly sets DHCP options that will be
advertised.

The first argument is the option value, [according to the RFC 2132][RFC2132]
(or this [IANA page][ianaopts]) directly or the enum `option:router`. The
seccond argument is the value that will be advertised. In this case
`10.0.0.254`, which is the IP address of the ISP's modem/routers.

Finally, the `dhcp-authoritative` is to indicate that this is the main DHCP
server of the network.

> For more options on how to configure the DHCP, [consult the DNSMasq
> manual page][manual].

After writing and saving the configuration file, it's necessary to restart the
dnsmasq service and everything is ready to be used:

{% highlight terminal %}
$ sudo service dnsmasq restart
{% endhighlight %}

### Fixed Address Hosts ###

Since there is a limited IP address range being distributed with limited lease
time, it's inevitable that sooner or later the address advertised for a certain
device will be different. 


Usually this is not a problem, but in certain situations it's important to
define fixed IP addresses for certain elements. The most common cases in a
domestic network are file servers or printers, but they can also be necessary
for setting up NAT Port Forwarding.

To create a fixed address host we use the `dhcp-host` configuration:

{% highlight ini %}
# My Computer
dhcp-host=AA:BB:CC:DD:EE:FF,mycomputer,10.0.0.1,infinite

# Print Server
dhcp-host=00:11:22:33:44:55,printer,10.0.0.2,infinite
{% endhighlight %}

In this case, the `10.0.0.1` is reserved for the device with
`AA:BB:CC:DD:EE:FF` MAC address.

Notice that the lease time used is **infinite**, meaning that the host will
never change the IP address. It's also possible to define limited leases, but
it doesn't make much sense.

> Also notice that the addresses are not in the other DHCP range.

Something that is very interesting in the DNSMasq is that by specifying a
hostname for a fixed address device, it will be already included in the DNS
service. Following the example, next time it's necessary to connect to the
printer from a PC, instead of pointing to `10.0.0.2`, it's possible to access
`printer.raspberry.local` (remember the domain name configured in the
beginning):

{% highlight terminal %}
~$ host printer.raspberry.local
printer.raspberry.local has address 10.0.0.2

~$ ping printer.raspberry.local
PING printer (10.0.0.2) 56(84) bytes of data.
64 bytes from printer (10.0.0.2): icmp_seq=1 ttl=64 time=1.45 ms
64 bytes from printer (10.0.0.2): icmp_seq=2 ttl=64 time=0.144 ms
 --- printer ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.144/0.799/1.454/0.655 ms
{% endhighlight %}

## References ##

1. [DNSMasq Man Page](http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)
2. [Hey Stephen Wood's English Tutorial](http://www.heystephenwood.com/2013/06/use-your-raspberry-pi-as-dns-cache-to.html) (Focused on DNS Caching)
3. [Raspberry Pi Forum Post](http://www.raspberrypi.org/forums/viewtopic.php?t=46154)

[dnsmasq]: http://www.thekelleys.org.uk/dnsmasq/doc.html
[manual]: http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html
[rasppi]: http://www.raspberrypi.org/
[openwrt]: https://openwrt.org/
[ddwrt]: https://www.dd-wrt.com/site/index
[raspbian]: http://www.raspbian.org/
[opendns]: http://www.opendns.com/home-internet-security/opendns-ip-addresses/
[googledns]: https://developers.google.com/speed/public-dns/
[rfc2132]: https://www.ietf.org/rfc/rfc2132.txt
[ianaopts]: http://www.iana.org/assignments/bootp-dhcp-parameters/bootp-dhcp-parameters.xhtml
