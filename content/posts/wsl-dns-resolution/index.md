---
title: Fixing WSL2 DNS Resolution
description: A quick workaround for DNS resolution issues in WSL2
date: 2021-01-04
tags: [Linux, WSL]
---

Recently, I've switched to using WSL2 on my Windows machine. It works quite well and has a ton of performance
improvements over WSL, but there's a major annoyance with the way DNS is configured through the Hyper-V Virtual
Ethernet adapter. VPN users beware! There are known DNS leak issues with WSL2 (documented 
[here](https://mullvad.net/en/blog/2020/9/30/linux-under-wsl2-can-be-leaking/)) that could compromise the privacy
of your web traffic.

This isn't that big of an issue for me, however, since I don't really use WSL2 for anything I'd consider critical
(mostly just using network connectivity for installing packages, pushing to git repos, etc). The bigger problem is that
my DNS queries will fail to resolve at all while I have active VPN connections. Here's a quick workaround, documented
below for my own future reference and anyone else who might be having this same issue:


1. Create a file at the following location: 
```
/etc/wsl.conf
```
2. It should have the following contents:
```
[network]
generateResolvConf = false
```
3. Run `wsl --shutdown` in a `cmd` window and restart WSL2.
4. Create another file: `/etc/resolv.conf`, replacing it if it already exists.
5. It should have the following contents:
```
nameserver YOUR_DNS_SERVER_HERE
```
6. Repeat step 3. Now WSL2 should be using the DNS server you've specified!


Note: For step 5, replace `YOUR_DNS_SERVER_HERE` with your desired DNS server. I like to use
[Adguard DNS servers](https://adguard.com/en/adguard-dns/overview.html#instruction), but some other options are
`8.8.8.8` for Google DNS, `1.1.1.1` for CloudFlare DNS, and `45.90.28.192` for NextDNS.


Credit: [https://github.com/microsoft/WSL/issues/4285#issuecomment-522201021](https://github.com/microsoft/WSL/issues/4285#issuecomment-522201021)
