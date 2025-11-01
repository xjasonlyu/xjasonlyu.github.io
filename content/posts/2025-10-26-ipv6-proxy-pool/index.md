---
title: "IPv6 Proxy Pool"
date: 2025-10-26T14:27:28-04:00
Description: ""
Tags: [linux, proxy, ipv6, proxy pool, network]
Categories: [Proxy, IPv6, Network, Linux, OS]
DisableComments: false
toc: true
---

This article is a step-by-step guide (and personal note) on how to configure an IPv6 proxy pool.

It's based on practical experiments with Debian/Ubuntu systems, with special thanks to [zu1k](https://github.com/zu1k) and [missuo](https://github.com/missuo) for their excellent base guides and references.

## How It Works

**Assumptions:**

- We have a dedicated IPv6 subnet assigned by the provider - typically `/64` (a larger prefix such as `/54` also works).
- The operating system is **Debian** or **Ubuntu**.

### Route and Bind

Once we have the IPv6 subnet, we need to add a route for it.

Obviously, we don't want to add every address in this `/64` subnet individually - that would quickly exhaust system resources. Instead, we can use the `ip route add local` trick to tell the kernel that all addresses in `2606:4700:4700::/64` are locally "owned" by the `ens3` interface.

> The `dev ens3` doesn't literally assign ownership of all these addresses to the interface; it just specifies the device used for routing lookups. Packets to that prefix will be delivered locally regardless of whether any individual address is configured.

```sh
ip route add local 2606:4700:4700::/64 dev ens3
```

To verify that the route has been added successfully, run:

```sh
ip -6 route show table local
```

Next, we need to enable the `ip_nonlocal_bind` option in the Linux kernel. This allows processes to bind to IP addresses that are not explicitly assigned to a local interface. Otherwise, any attempt to bind to such addresses will fail.

```sh
sysctl net.ipv6.ip_nonlocal_bind=1
```

### Configure NDP

If the VPS shares the same Layer 2 (VLAN) network with other hosts, IPv6 Neighbor Discovery (ND) is used to resolve MAC addresses (similar to ARP in IPv4).

Because we didn't explicitly assign every address in the `/64` subnet to the interface, the kernel won't respond to Neighbor Solicitation messages for those unassigned addresses.

To handle these requests, we can use an Neighbor Discovery Proxy (NDP) to reply on behalf of the kernel.

> However, if the VPS is **directly routed** - meaning the upstream router already knows your MAC address and sends traffic for the entire prefix straight to you - then no NDP proxy is required.
>
> We can check whether an NDP proxy is needed by capturing Neighbor Solicitation (ICMPv6 type 135) messages:
>
> ```sh
> tcpdump -i <eth0> icmp6 and 'ip6[40] == 135'
> ```
>
> If we see such packets coming from the upstream router, we'll need to run an NDP proxy.
> If not, we can simply assign the IPv6 subnet to `dev lo`, since the prefix is directly routed to our host.

To set up NDP proxying, install ndppd:

```sh
apt install ndppd
```

Then edit `/etc/ndppd.conf` (replace the interface name `ens3` and the prefix with actual values):

```text
route-ttl 30000
proxy ens3 {
    router no
    timeout 500
    ttl 30000
    rule 2606:4700:4700::/64 {
        static
    }
}
```

Finally, enable and restart the service:

```sh
systemctl enable ndppd.service
systemctl restart ndppd.service
```

> **NOTE**: If your setup relies on NDP, avoid using subnets larger than `/120`. Otherwise, the system may generate too many neighbor entries, which can flood the ND table and cause performance issues.

### Verify It Works

To confirm that the setup is functional, we can use `curl --interface` to bind outbound connections to specific IPv6 addresses within our `/64` subnet.  
If everything is configured correctly, each request should appear to come from the specified address.

```sh
$ curl --interface 2606:4700:4700::a ipv6.ip.sb
2606:4700:4700::a
$ curl --interface 2606:4700:4700::b ipv6.ip.sb
2606:4700:4700::b
```

## Docker Setup

Personally, I don't like exposing or hosting services directly on my host machine, so I use Docker for a bit of isolation and easier management.

When it comes to making an IPv6 proxy pool inside Docker, the main challenge is networking - specifically, how containers get their IPv6 connectivity (for a broad range of IPv6 addresses).

Docker supports several main network modes:

- bridge
- host
- none
- macvlan
- ipvlan

The bridge network doesn't work for our use case. It creates a NATed environment behind the host network, so no matter what IPv6 address you assign inside the container, the actual outbound IP on the public internet will still be your host's IP.

You _could_ mess around with host routing rules to make it work, but that's unnecessary and goes against the whole purpose of this setup.

The none network - well, it's literally "no network", so that's not what we want either.

MacVLAN is actually a great fit for this kind of setup. However, most VPS or IDC providers don't allow creating new MAC addresses in their local network. Doing so might trigger security alerts or even violate their terms of service. If you're running this in your own physical network like a HomeLab, though, that's perfectly fine - MacVLAN would be the best and cleanest option AFAIK.

That leaves us with two remaining options: host and ipvlan. Both can work, but each has its own trade-offs.

### Host

Using the **host** network mode is basically the same as setting everything up directly on the host itself.
With the following Docker Compose setup, you can enable the host network and allow non-local IPv6 bindings via `sysctls`:

```yaml
services:
  proxy-pool:
    cap_add:
      - NET_ADMIN
    sysctls:
      net.ipv6.ip_nonlocal_bind: "1"
    network_mode: host
    ...
```

You might need a small custom or entrypoint script to set up the ip route add local rule before starting the actual proxy process.

The main drawback of using the host mode is that it’s not fully isolated - any route changes made inside the container will directly affect the host's network namespace.
So personally, I don't really see the point of running it in host mode unless you have other specific needs. If you're going to do that, you might as well just run it directly on the host.

### IPVlan

I initially thought **IPvlan** could be the ultimate solution - it

- shares the same MAC address as the parent interface on the host, and
- provides proper network isolation between containers and the host.

However, things didn't go as smoothly as I expected.

After setting everything up with the IPvlan network (including enabling non-local bind and the ip route local trick),
I noticed something odd: I still couldn't send requests using IP addresses that weren't explicitly assigned to the interface.

So I traced the issue further using `tcpdump`. Outgoing packets were sent successfully, and reply packets were indeed routed back. I could even see _Neighbor Solicitation_ (NS) messages on the network asking for the IP's MAC address - and I could confirm those packets reached the host interface. But they never made it into the container’s network namespace.

So I started digging into the IPvlan driver implementation.

#### How IPvlan Resolves Incoming Packets

Inside the kernel source, there's a function called `ipvlan_addr_lookup` which looks up the associated port (container) for a given IP address when handling a packet via `ipvlan_handle_mode_l3`.

```c
struct ipvl_addr *ipvlan_addr_lookup(struct ipvl_port *port, void *lyr3h,
				     int addr_type, bool use_dest)
{
	struct ipvl_addr *addr = NULL;

	switch (addr_type) {
#if IS_ENABLED(CONFIG_IPV6)
	case IPVL_IPV6: {
		struct ipv6hdr *ip6h;
		struct in6_addr *i6addr;

		ip6h = (struct ipv6hdr *)lyr3h;
		i6addr = use_dest ? &ip6h->daddr : &ip6h->saddr;
		addr = ipvlan_ht_addr_lookup(port, i6addr, true);
		break;
	}
```

This lookup uses a hash table, implemented as follows:

```c
static struct ipvl_addr *ipvlan_ht_addr_lookup(const struct ipvl_port *port,
					       const void *iaddr, bool is_v6)
{
	struct ipvl_addr *addr;
	u8 hash;

	hash = is_v6 ? ipvlan_get_v6_hash(iaddr) :
	       ipvlan_get_v4_hash(iaddr);
	hlist_for_each_entry_rcu(addr, &port->hlhead[hash], hlnode)
		if (addr_equal(is_v6, addr, iaddr))
			return addr;
	return NULL;
}
```

#### When Does This Hash Table Get Populated

It's populated either when the IPvlan device is opened, or when an address is added to the interface.

When opened:

```c
static int ipvlan_open(struct net_device *dev)
{
	struct ipvl_dev *ipvlan = netdev_priv(dev);
	struct ipvl_addr *addr;

	if (ipvlan->port->mode == IPVLAN_MODE_L3 ||
	    ipvlan->port->mode == IPVLAN_MODE_L3S)
		dev->flags |= IFF_NOARP;
	else
		dev->flags &= ~IFF_NOARP;

	rcu_read_lock();
	list_for_each_entry_rcu(addr, &ipvlan->addrs, anode)
		ipvlan_ht_addr_add(ipvlan, addr);
	rcu_read_unlock();

	return 0;
}
```

When an IPv6 address is added or removed:

```c
static int ipvlan_addr6_event(struct notifier_block *unused,
			      unsigned long event, void *ptr)
{
	struct inet6_ifaddr *if6 = (struct inet6_ifaddr *)ptr;
	struct net_device *dev = (struct net_device *)if6->idev->dev;
	struct ipvl_dev *ipvlan = netdev_priv(dev);

	if (!ipvlan_is_valid_dev(dev))
		return NOTIFY_DONE;

	switch (event) {
	case NETDEV_UP:
		if (ipvlan_add_addr6(ipvlan, &if6->addr))
			return NOTIFY_BAD;
		break;

	case NETDEV_DOWN:
		ipvlan_del_addr6(ipvlan, &if6->addr);
		break;
	}

	return NOTIFY_OK;
}
```

#### The Dead End

And here's the problem:

Only IPs that are explicitly assigned to the IPvlan interface get inserted into that hash table. When a packet arrives, the kernel looks it up there - if it doesn't exist, it drops or ignores it.

Even if we tried to fake-add addresses manually, maintaining that table for an entire `/64` network would be unmanageable. A `/64` prefix has 18 quintillion addresses, which is far beyond what's feasible for IPvlan's current implementation.

Sure, in theory we could modify the kernel driver, recompile it, or hook something with eBPF to bypass that lookup - but the added complexity just isn't worth it for this use case.

## Conclusion

TBH, I don't have anything to conclude here. 💁

## References

- <https://missuo.me/posts/ipv6-proxy-pool/>
- <https://zu1k.com/posts/tutorials/http-proxy-ipv6-pool/>
- <https://www.alibabacloud.com/blog/599797>
- <https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ipvlan/ipvlan_main.c>
