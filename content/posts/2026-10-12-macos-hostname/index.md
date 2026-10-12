---
title: "Tracing a Changing Hostname on macOS"
date: 2026-10-12T14:00:00-04:00
Description: "Tracing an unexpected macOS hostname from an iTerm2 title to DHCP Option 12, Private Wi-Fi Address, and Apple's bootp source code."
Tags: [macos, dhcp, hostname, wifi]
Categories: [macOS, Network]
DisableComments: false
toc: true
---

## Motivation

One day, I noticed that the title in my iTerm2 window had changed:

```text
jason@Jasons-MacBook-Pro  ->  jason@Mac
```

The first title looked familiar: `Jasons-MacBook-Pro` was the hostname I expected to see. `Mac`, however, was new. I definitely didn't remember renaming the machine to something that generic.

Since the shell title included the current hostname, the change meant that `hostname` had started returning `Mac`. It didn't tell me what changed it.

So I started from the names macOS already knew about.

## Configured Name vs. Live Name

macOS has several related names which are easy to mix up:

```console
$ scutil --get ComputerName
Jason's MacBook Pro

$ scutil --get LocalHostName
Jasons-MacBook-Pro

$ scutil --get HostName
HostName: not set
```

Apple describes `ComputerName` as the human-readable name and `LocalHostName` as the Bonjour-compatible local network name. Normally the latter is derived by replacing spaces with hyphens and appending `.local` when used on the network.

But the live BSD hostname was different:

```console
$ hostname
Mac
```

Because `HostName` was not explicitly set, this value could be supplied dynamically by the network environment. That explained why the iTerm2 title could differ from both `ComputerName` and `LocalHostName`: it reflected the live result of `hostname`.

The old iTerm2 title showed that the live hostname had previously resembled `LocalHostName`. I hadn't captured the system or network state at the moment the title changed, so I couldn't yet say why or exactly when it was replaced by `Mac`.

Next I needed to see where `Mac` came from.

## Following the Name to the Bell Giga Hub

My Mac was using a Bell Giga Hub as its DHCP server and local DNS resolver. A reverse lookup of the Mac's current Wi-Fi address returned the same name:

```console
$ dig @192.168.2.1 -x "$(ipconfig getifaddr en0)" +short
Mac.
```

This connected the live hostname to the router's local DNS, but it still didn't establish direction. The router might have invented `Mac`, or the Mac might have announced it and the router simply published it as a PTR record.

The quickest way to distinguish those possibilities was to look at DHCP itself.

## Capturing DHCP Option 12

DHCP Option 12 is the client Host Name option. I started a capture on the Wi-Fi interface, then renewed the connection:

```sh
sudo tcpdump -i en0 -n -s 0 -vvv \
  'udp port 67 or udp port 68'
```

With **Private Wi-Fi Address enabled**, the relevant part of the DHCP request looked like this (MAC addresses have been redacted):

```text
Request from <PRIVATE-WIFI-MAC>
    DHCP-Message (53), length 1: Discover
    Client-ID (61), length 7: ether <PRIVATE-WIFI-MAC>
    Hostname (12), length 3: "Mac"
```

That answered the direction question. `Mac` was already present in the request sent by the laptop, before the Giga Hub offered an address. The router wasn't the original source of that string.

It also suggested another experiment. The client was using a private MAC address, and macOS was deliberately sending the generic name `Mac`. What would happen if I disabled Private Wi-Fi Address for this network?

## Private Wi-Fi Address Off: A Third Name

After switching Private Wi-Fi Address off and reconnecting, several things changed together. Instead of returning to the full `Jasons-MacBook-Pro`, macOS introduced a third name: `Jasons-MBP`.

The DHCP client now used the hardware MAC address and sent a different Option 12:

```text
13:56:33.530549 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request
    Client-Ethernet-Address <HARDWARE-WIFI-MAC>
    DHCP-Message (53), length 1: Discover
    Client-ID (61), length 7: ether <HARDWARE-WIFI-MAC>
    Hostname (12), length 10: "Jasons-MBP"
```

The Giga Hub then returned the same name from its local reverse DNS:

```console
$ dig @192.168.2.1 -x "$(ipconfig getifaddr en0)" +short
Jasons-MBP.
```

And the live hostname followed:

```console
$ hostname
Jasons-MBP
```

The observed states were now quite consistent:

| Private Wi-Fi Address | Wi-Fi MAC             | DHCP Option 12 | Bell PTR      | `hostname`   |
| --------------------- | --------------------- | -------------- | ------------- | ------------ |
| Enabled               | `<PRIVATE-WIFI-MAC>`  | `Mac`          | `Mac.`        | `Mac`        |
| Disabled              | `<HARDWARE-WIFI-MAC>` | `Jasons-MBP`   | `Jasons-MBP.` | `Jasons-MBP` |

This didn't mean the MAC address itself somehow encoded the hostname. It showed that macOS selected a different DHCP identity when its Wi-Fi privacy mode changed.

## Was Jasons-MBP Stored Somewhere?

There was still one annoying question: where did `Jasons-MBP` come from? I had never configured that name.

First I checked the SMB/NetBIOS name:

```console
$ defaults read /Library/Preferences/SystemConfiguration/com.apple.smb.server \
    NetBIOSName 2>/dev/null
MAC
```

That was a separate naming mechanism and didn't match the DHCP value anyway.

Then I searched the persistent SystemConfiguration preferences:

```console
$ plutil -p /Library/Preferences/SystemConfiguration/preferences.plist \
    | grep -i -C 4 -E 'Jasons-MBP|Jasons-MacBook-Pro|DHCPClientID'
      "HostNames" => {
        "LocalHostName" => "Jasons-MacBook-Pro"
      }
    "System" => {
      "ComputerName" => "Jason's MacBook Pro"
```

The dynamic store showed the same configured values:

```console
$ printf 'show Setup:/System\nshow Setup:/Network/HostNames\n' | scutil
<dictionary> {
  ComputerName : Jason's MacBook Pro
  ComputerNameEncoding : 0
}
<dictionary> {
  LocalHostName : Jasons-MacBook-Pro
}
```

`Jasons-MBP` wasn't there. It wasn't in NVRAM either:

```console
$ nvram -p | grep -iE 'computer-name|Jasons-MBP|hostname'
```

No output.

So this didn't look like a forgotten manual setting. More likely, the DHCP client was generating the shorter name at runtime.

## Asking configd

The unified log confirmed that `configd` was both supplying the name in DHCP and later setting the live hostname:

```sh
log show --last 30d --style compact \
  --predicate 'process == "configd" AND eventMessage CONTAINS[c] "Jasons-MBP"'
```

Representative lines from the output:

```text
2026-10-12 13:52:48.815 Df configd[...] DHCP en0: supplying hostname 'Jasons-MBP'
2026-10-12 13:52:53.855 Df configd[...] setting hostname to "Jasons-MBP"
2026-10-12 13:56:33.538 Df configd[...] DHCP en0: supplying hostname 'Jasons-MBP'
2026-10-12 13:56:38.464 Df configd[...] setting hostname to "Jasons-MBP"
```

This ruled out the Giga Hub as the creator of `Jasons-MBP`: macOS had chosen the string before putting it into the DHCP request.

The last step was to find the code which chose it.

## Apple's bootp Source

Apple publishes the source for its `bootp` project. The version I inspected was the public [`bootp-534.100.6`](https://github.com/apple-oss-distributions/bootp/tree/bootp-534.100.6) tag from April 2026.

One caveat before going further: I didn't prove that the exact `configd` binary on this Mac was built byte-for-byte from that tag. Apple may ship code that differs from, or is newer than, the latest public source drop. Still, the published control flow, log strings, and final output all match the observed behavior, so it is strong evidence for the mechanism rather than a claim about exact binary provenance.

### Privacy Mode Chooses What to Send

In [`add_host_name()`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/IPConfiguration.bproj/dhcp.c#L467-L507), the DHCP client makes one of three choices:

1. If privacy is not required, use the normal DHCP hostname.
2. If privacy is required but sharing the device type is allowed, use the device type.
3. Otherwise, omit the hostname.

The related [`ServiceIsPrivacyRequired()`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/IPConfiguration.bproj/ipconfigd.c#L5424-L5460) treats a private Wi-Fi link address as requiring privacy. The device-type path reads the product type and keeps the alphabetic prefix. A value beginning with something like `Mac...` therefore becomes `Mac`.

That lines up with the first capture: private address enabled, DHCP Option 12 set to `Mac`.

### The Normal DHCP Hostname Path

When privacy is not required, [`computer_name_update()`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/IPConfiguration.bproj/ipconfigd.c#L1252-L1293) tries the names in this order:

1. Use `ComputerName` if it is already a valid DNS name.
2. Otherwise, use `LocalHostName` if it is valid.
3. Shorten the local hostname if necessary.

My `ComputerName`, `Jason's MacBook Pro`, contains spaces and an apostrophe, so it doesn't take the first path. The fallback is:

```text
Jasons-MacBook-Pro
```

That is 18 characters. The source uses a default maximum of 15 for this local-hostname-derived DHCP value.

The shortening happens in [`myCopyDHCPSafeHostName()`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/bootplib/cfutil.c#L920-L975). Before doing a generic truncation, it applies a case-insensitive list of product-name replacements, including:

```c
{ CFSTR("macbook-pro"), CFSTR("MBP") }
```

So the transformation is straightforward:

```text
Jasons-MacBook-Pro
       └─────────┘
       macbook-pro  ->  MBP

Jasons-MBP
```

The same source file even includes a test case mapping `jane-macbook-pro` to `jane-MBP` with a maximum length of 15.

There was no hidden `Jasons-MBP` setting after all. It was calculated from `Jasons-MacBook-Pro` by the DHCP hostname shortening function.

## Putting It All Together

With Private Wi-Fi Address enabled, the observed path was:

```text
private Wi-Fi address
  -> DHCP privacy path
  -> device type "Mac" in Option 12
  -> Bell Giga Hub publishes Mac. as the local PTR
  -> the live hostname becomes Mac
  -> iTerm2 displays jason@Mac
```

With it disabled:

```text
hardware Wi-Fi address
  -> normal DHCP hostname path
  -> ComputerName rejected as a DNS name
  -> LocalHostName Jasons-MacBook-Pro
  -> macbook-pro shortened to MBP
  -> Jasons-MBP in Option 12
  -> Bell Giga Hub publishes Jasons-MBP. as the local PTR
  -> the live hostname becomes Jasons-MBP
  -> iTerm2 displays jason@Jasons-MBP
```

The packet captures establish what the Mac sent, the Giga Hub lookup shows what the router published, and the `configd` logs show the hostname transitions. The public Apple source explains the two strings macOS chose. I haven't separately proven every internal step by which `IPMonitor` consumes the network's naming information, but that uncertainty doesn't affect the origin of either DHCP hostname.

The title change was therefore never really an iTerm2 problem. It simply exposed a change in the system hostname. The controlled tests explain the live `Mac` state and the later `Jasons-MBP` state; the earlier `Jasons-MacBook-Pro` state predates those captures, so the exact network state at that moment remains unverified.

Still, that small change in a terminal title was the first visible trace of a DHCP privacy decision deep inside macOS—and eventually led to a hostname shortening rule hidden in Apple's open-source network code.

## References

- [Apple: Change your computer's name or local hostname on Mac](https://support.apple.com/guide/mac-help/change-your-computers-name-or-local-hostname-mchlp2322/mac)
- [Apple: Use private Wi-Fi addresses on Apple devices](https://support.apple.com/102509)
- [RFC 2132, Section 3.14: Host Name Option](https://datatracker.ietf.org/doc/html/rfc2132#section-3.14)
- [Apple Open Source: bootp-534.100.6](https://github.com/apple-oss-distributions/bootp/tree/bootp-534.100.6)
- [`add_host_name()` in `dhcp.c`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/IPConfiguration.bproj/dhcp.c#L467-L507)
- [`computer_name_update()` and `get_dhcp_hostname()` in `ipconfigd.c`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/IPConfiguration.bproj/ipconfigd.c#L1240-L1293)
- [`myCopyDHCPSafeHostName()` in `cfutil.c`](https://github.com/apple-oss-distributions/bootp/blob/63cb751ca6a4e148704a701090481fd9a895dda0/bootplib/cfutil.c#L920-L1000)
