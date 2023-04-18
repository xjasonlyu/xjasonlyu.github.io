---
title: "SO_BINDTODEVICE: How It Works in Linux"
date: 2020-12-03T16:14:24+08:00
Description: ""
Tags: [linux, bindtodevice, socketopt, vpn]
Categories: [Linux, Network, OS]
DisableComments: false
toc: true
aliases: [so_bindtodevice]
---

### 起因

因为写tun2socks的缘故，近来断断续续搜索+研究了一下下如何让socket流量只从特定的interface发出而不走路由表，这样就可以防止特定流量经过tun接口然后无限循环。

### 策略路由+bind

在Linux下其实方法挺多的，比如可以用策略路由（`policy routing`），先用`bind`绑定特定的接口地址，然后根据源地址配置用双路由表即可。

~~这里不得不吐槽Windows下既不能绑接口也没有策略路由，可能是我还没找到，反正目前没想好怎么解决。~~

`bind`函数的作用只是让发送包的源地址是你绑定的接口地址，但是在匹配路由表的时候该怎么走还是怎么走（匹配路由只看目的地址），而且还可能因为源地址不对直接回不来了。

### SO_BINDTODEVICE

当然，在Linux下有个更好的解决方案：`SO_BINDTODEVICE`，这与`bind`是不同的。

根据文档，

> Bind this socket to a particular device like "eth0", as specified in the passed interface name. If the name is an empty string or the option length is zero, the socket device binding is removed. The passed option is a variable-length null-terminated interface name string with the maximum size of IFNAMSIZ. If a socket is bound to an interface, only packets received from that particular interface are processed by the socket. Note that this only works for some socket types, particularly AF_INET sockets. It is not supported for packet sockets (use normal bind(2) there).
> Before Linux 3.8, this socket option could be set, but could not retrieved with getsockopt(2). Since Linux 3.8, it is readable. The optlen argument should contain the buffer size available to receive the device name and is recommended to be IFNAMSZ bytes. The real device name length is reported back in the optlen argument.

嗯，看的似懂非懂，而且没找到跟路由有关的信息。

然后再查资料，终于看到

> The bind() system call is frequently misunderstood. It is used to bind to a particular IP address. Only packets destined to that IP address will be received, and any transmitted packets will carry that IP address as their source. bind() does not control anything about the routing of transmitted packets. So for example, if you bound to the IP address of eth0 but you send a packet to a destination where the kernel's best route goes out eth1, it will happily send the packet out eth1 with the source IP address of eth0. This is perfectly valid for TCP/IP, where packets can traverse unrelated networks on their way to the destination.

`SO_BINDTODEVICE`会保证该socket的所有流量都会从该接口出去，但依旧会有查询路由表的操作，只是会跳过不匹配的接口，如果路由表找不到就会network unreachable。

同样类似的还有`SO_DONTROUTE`选项，并不是不路由的意思，而且依旧要查询路由表，只是不走网关，作用仅在LINK范围。

所以可以配置两个不同优先级的默认路由，这样就可以在绑定接口后走不同的路由了。同样的，可以拿多端口来做负载均衡。

然后注意要关闭rp_filter内核参数，虽然Linux内核是默认禁用的，但是有的发行版为了安全会打开，不然抓包看数据都有，但是socket拿不到。

```sh
sysctl -w net.ipv4.conf.eth0.rp_filter=0
```

最后拿Docker跑了个双接口的容器验证了一下，具体的细节就不贴了

### 补充（2021-08-27）

除了BINDTODEVICE以外也有其他方式解决

- [cgroup + namespace](https://www.wireguard.com/netns/)
- 魔改Kernel（直接pass原Interface）

### 参考

- [SO_DONTROUTE和SO_BINDTODEVICE的深层次分析][1]
- [SO_BINDTODEVICE socket option for Linux 2.0.30+][2]
- [Code Snippet: SO_BINDTODEVICE][3]

[1]: https://blog.51cto.com/dog250/1271769
[2]: http://ftp.slackware-brasil.com.br/slackware-3.5/docs/linux-2.0.34/networking/so_bindtodevice.txt
[3]: https://codingrelic.geekhold.com/2009/10/code-snippet-sobindtodevice.html
