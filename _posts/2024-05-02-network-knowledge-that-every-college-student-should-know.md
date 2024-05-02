---
layout: post
title: Network knowledge that every college student should know
date: 2024-05-02 16:23 -0700
categories: [network]
tags: [network]
---

## Router, LAN, WAN, NAT

Each router has a private ip address and a public ip address.

Router will assign private ip address to the devices behind its local area
network (LAN). Use a home LAN as an example. Suppose your router has a private
ip address 192.168.0.1 and public ip address 99.99.99.99. The public ip address
is determined by the internet provider you use, such as AT&T. The private ip
address can be whatever the router manufacture decides, but it is usually
192.168.0.1. You can google why 192.168.0.1 is the most used private ip address
for a router :). For each device in this LAN, like your home desktop, it will
be assigned a private address in the same subnet like 192.168.0.22. When your
desktop needs to connect to a website on internet, say 88.88.88.88, it sends a
web packet like `{from:192.168.0.22, to: 88.88.88.88, data:...}` to router,
then router transforms this packet to `{from: 99.99.99.99, to: 88.88.88.88}`,
and sends it to internet. Meanwhile, router adds a record

```
192.168.0.22: 88.88.88.88
```

to its NAT (network address translation) table, which means "I am communicating
to 88.88.88.88 on behalf of 192.168.0.22". With NAT, router knows which device
it should forward the packets received from internet. The details here are not
restrict. Just get my idea.

Usually, the backside of the router provides login info to change router
settings. The user name and password is usually `admin` and `admin`. Also, you
can find the router private ip address from your phone or laptop and directly
typing it in your browser will open the setting page.

As said, router is a device to connect your devices in a LAN to WAN (wide are
network), i.e., internet. It must has two ends. One end connects LAN devices,
and the other end connects to WAN device. On the back of a router, you can find
two kinds of ports. One is for LAN. You connect your home laptop to this port
using a wire. The other port is WAN and connects to a modem.

![router](/assets/images/router.jpeg)

### ARP, MAC

As said in the above section, NAT plays an important role connecting LAN and
WAN. That happens at network level, i.e., IP level. What under it is data link
level. This level deals with MAC (media access control) address. How does
router know which physical device to forward traffic to given a private ip
address? This is called ARP (Address Resolution Protocol). Basically, It is a
cache that maps IP address to MAC address. See more details
[here](https://lartc.org/howto/lartc.iproute2.arp.html).

### Routing table

### Router vs gateway, default gateway vs network switch

First router and network switch are very similar. Probably network switch
usually refers to a "router" with much better performance.

I still cannot figure out the difference between a gateway and router. From
what I read, gateway has ability to translate protocols between two networks.

### Network interface

First some terminology and abbreviations.

- NIC: network interface controller, or called it another way, network
  interface card.
-

### virtual network interface

veth: Virtual Ethernet Device. See its
[man page](https://man7.org/linux/man-pages/man4/veth.4.html)

### netfilters, iptables

TODO: I know little about netfilers. I should read more about this subject.

There are multiple iptables. One is filter table, core for firewall rule. The
other is NAT table, kind of network proxy built inside Linux kernel.

### References

- CIDR
  https://www.digitalocean.com/community/tutorials/understanding-ip-addresses-subnets-and-cidr-notation-for-networking

## Websocket

Websocket requires two headers `Connection` and `Upgrade` at the initial
handshake stage. While these two are hop-by-hop headers, so it will affect
websocket proxy.

See how Nginx deals with websocket https://www.nginx.com/blog/websocket-nginx

## DNS

- DNS: domain name system. Refer to this system as whole
- DNS lookup: the process that translate a domain name to an ip address.
- DNS records: records in Authoritative server. Ex: A record, CNAME record,
  etc.

See [cloudflare doc](https://www.cloudflare.com/learning/dns/what-is-dns/) and
[RFC-1034](https://datatracker.ietf.org/doc/html/rfc1034)

### DNS resolver

Read the [wiki](https://en.wikipedia.org/wiki/Resolv.conf) of
`/etc/resolv.conf`. It defines the name server and list of
[searches](https://superuser.com/questions/570082/in-etc-resolv-conf-what-exactly-does-the-search-configuration-option-do)

### DNS in kubernetes

Check this
[post](https://aws.amazon.com/premiumsupport/knowledge-center/eks-dns-failure/)
