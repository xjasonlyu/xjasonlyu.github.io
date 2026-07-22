---
title: "SO_BINDTODEVICE Socket Option in Linux"
date: 2020-12-03T16:14:24+08:00
Description: ""
Tags: [linux, bindtodevice, socketopt, vpn]
Categories: [Linux, Network, OS]
DisableComments: false
toc: true
aliases: [so_bindtodevice]
---

### Motivation

While working on tun2socks, I spent some time on and off looking into how to make socket traffic leave through a specific interface instead of following the routing table. This would prevent certain traffic from going through the TUN interface and getting stuck in an infinite loop.

### Policy Routing + bind

There are actually quite a few ways to do this on Linux. One option is policy routing: use `bind` to bind to a specific interface address, then configure two routing tables based on the source address.

~~I have to complain about Windows here: it supports neither binding to an interface nor policy routing. Maybe I just haven't found the right way yet, but for now I have no idea how to solve this there.~~

The `bind` function only makes the bound interface address the source address of outgoing packets. Route lookup still works exactly as before (it only looks at the destination address), and the reply might not even make it back if the source address is wrong.

### SO_BINDTODEVICE

Of course, Linux has a better solution: `SO_BINDTODEVICE`, which is different from `bind`.

According to the documentation:

> Bind this socket to a particular device like "eth0", as specified in the passed interface name. If the name is an empty string or the option length is zero, the socket device binding is removed. The passed option is a variable-length null-terminated interface name string with the maximum size of IFNAMSIZ. If a socket is bound to an interface, only packets received from that particular interface are processed by the socket. Note that this only works for some socket types, particularly AF_INET sockets. It is not supported for packet sockets (use normal bind(2) there).
> Before Linux 3.8, this socket option could be set, but could not retrieved with getsockopt(2). Since Linux 3.8, it is readable. The optlen argument should contain the buffer size available to receive the device name and is recommended to be IFNAMSZ bytes. The real device name length is reported back in the optlen argument.

Hmm. I sort of understood it, but couldn't find anything about routing.

After digging around some more, I finally found this:

> The bind() system call is frequently misunderstood. It is used to bind to a particular IP address. Only packets destined to that IP address will be received, and any transmitted packets will carry that IP address as their source. bind() does not control anything about the routing of transmitted packets. So for example, if you bound to the IP address of eth0 but you send a packet to a destination where the kernel's best route goes out eth1, it will happily send the packet out eth1 with the source IP address of eth0. This is perfectly valid for TCP/IP, where packets can traverse unrelated networks on their way to the destination.

`SO_BINDTODEVICE` ensures that all traffic from the socket goes through that interface. The routing table is still consulted, but routes for other interfaces are skipped; if no matching route can be found, the result is `network unreachable`.

There is also a similar option called `SO_DONTROUTE`. Despite the name, it doesn't mean "don't route," and the routing table is still consulted. It only avoids gateways, so it works within link scope.

This means we can configure two default routes with different priorities and use different routes after binding to an interface. The same idea can also be used with multiple ports for load balancing.

Also, remember to disable the `rp_filter` kernel parameter. The Linux kernel disables it by default, but some distributions enable it for security. Otherwise, packet captures will show that the data is there, but the socket won't receive it.

```sh
sysctl -w net.ipv4.conf.eth0.rp_filter=0
```

Finally, I verified this with a dual-interface container running in Docker. I'll skip the details here.

### Update (2021-08-27)

There are other ways to solve this besides `BINDTODEVICE`:

- [cgroup + namespace](https://www.wireguard.com/netns/)
- Hack the kernel (pass through the original interface directly)

### References

- [An In-Depth Analysis of SO_DONTROUTE and SO_BINDTODEVICE][1]
- [SO_BINDTODEVICE socket option for Linux 2.0.30+][2]
- [Code Snippet: SO_BINDTODEVICE][3]

[1]: https://blog.51cto.com/dog250/1271769
[2]: http://ftp.slackware-brasil.com.br/slackware-3.5/docs/linux-2.0.34/networking/so_bindtodevice.txt
[3]: https://codingrelic.geekhold.com/2009/10/code-snippet-sobindtodevice.html
