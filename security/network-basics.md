# Network Basics for Security Recon

The foundational stuff that every recon tutorial assumes you already know. Written because I kept running commands without understanding why the numbers looked the way they did, and that gap eventually catches up with you.

## Private vs public IPs

Every device you use has at least two IP addresses associated with it: a **private IP** (visible only inside your local network) and a **public IP** (visible to the outside internet).

### Private IPs

Defined in a 1996 internet standard called RFC 1918. Three reserved ranges that any local network can use freely:

| Range | Common use |
|---|---|
| `10.0.0.0 – 10.255.255.255` | Large corporate networks, some ISPs |
| `172.16.0.0 – 172.31.255.255` | Some routers, less common at home |
| `192.168.0.0 – 192.168.255.255` | Almost every home router on earth |

These IPs are not routable on the public internet. They only exist inside the network they're assigned in. That's why every house on your street can have a laptop at `192.168.1.47` with no conflict. The numbers are reused because they never leave the building.

### Public IPs

The one address your network is known by from the outside. Assigned by your ISP. Globally unique. This is what the internet sees when you visit a website.

To find each:

```bash
# Private IP (your laptop on your local network)
ifconfig | grep "inet " | grep -v 127.0.0.1

# Public IP (your network's address to the outside world)
curl ifconfig.me
```

### Why this matters for recon

Every nmap scan, every Wireshark capture, every "what's on my network" exercise has been about private IPs. The public IP doesn't come up unless you're doing external-facing work like port forwarding, firewall rules, or pentesting a public-facing target.

Don't share your public IP carelessly. It geolocates to your rough area and identifies your network to anyone watching. Treat it like your home address: not catastrophic if it leaks, but not for public posts.

## How traffic flows between networks

When your laptop talks to a server on the internet, traffic doesn't go directly. It bounces through a translation layer called **NAT** (Network Address Translation) that lives in your router.

```
Your laptop (192.168.1.47)
  → Your router (NAT translates to your public IP)
  → Internet
  → Destination server
  → Response comes back
  → Your router (NAT translates back to 192.168.1.47)
  → Your laptop
```

Two different houses can both have laptops at `192.168.1.47` because each router handles its own translation. No conflict because the routers do the bookkeeping. This is also why you can't usually reach into someone else's home network from the outside. Their router doesn't know who should receive an unsolicited connection, so it drops it. That's a feature, not a bug.

## CIDR notation

Short for **Classless Inter-Domain Routing**. A way of writing network ranges as a single number.

The history: before 1993, IP networks were rigid sizes (Class A, B, C). Wasted huge chunks of address space. CIDR replaced the fixed classes with a flexible bit count.

The `/N` at the end of an IP range tells you how many bits of the address define the network vs the host. Fewer bits = bigger network.

| CIDR | Total IPs | Typical use |
|---|---|---|
| `/8` | 16,777,216 | ISP-scale network |
| `/16` | 65,536 | Large company |
| `/24` | 256 | Home network (most common) |
| `/30` | 4 | Point-to-point router link |
| `/32` | 1 | Single device |

For home recon, `/24` is almost always correct. If your laptop is at `192.168.1.47`, the network it lives on is `192.168.1.0/24`, which covers `192.168.1.0` through `192.168.1.255`.

## Finding your network range before scanning

Three pieces of info you need before running nmap on a local network:

```bash
# 1. Your private IP
ifconfig | grep "inet " | grep -v 127.0.0.1
# Returns something like: inet 192.168.1.47 ...

# 2. Your subnet mask (translates to CIDR)
ifconfig en0 | grep netmask
# Returns: netmask 0xffffff00 → this is /24

# 3. Your gateway (the router itself)
netstat -nr | grep default
# Returns: default 192.168.1.1 ...
```

From those three: your scan range is the first three octets of your IP plus `.0/24`. So `192.168.1.47` means scan `192.168.1.0/24`.

### Subnet mask to CIDR translation

The output of `ifconfig` shows the netmask in hex. Quick decoder for the common ones:

| Hex netmask | Dotted decimal | CIDR |
|---|---|---|
| `0xffffff00` | `255.255.255.0` | `/24` |
| `0xffff0000` | `255.255.0.0` | `/16` |
| `0xff000000` | `255.0.0.0` | `/8` |

You will see `/24` 99% of the time at home. Memorize that one.

## What the addresses inside a network mean

Inside any `/24` network, the IPs aren't randomly assigned. There's a pattern.

| IP | Role |
|---|---|
| `.0` | Network address (the name of the network itself, not a device) |
| `.1` | Usually the router/gateway |
| `.2` through `.254` | Available for devices |
| `.255` | Broadcast address (sends to all devices at once) |

So in `192.168.1.0/24`:
- `192.168.1.0` is the network label
- `192.168.1.1` is probably your router
- `192.168.1.255` is the broadcast address
- Everything in between is fair game for devices

This is why your router is almost always at `.1`. Convention.

## Two scan targets to know

**Scanning your own machine only:**
```bash
nmap localhost
```
Equivalent to `nmap 127.0.0.1`. Shows what ports your laptop has open. No authorization concerns because it's your own machine.

**Scanning your whole local network:**
```bash
sudo nmap -sn 192.168.1.0/24
```
The `-sn` is a ping scan (no port probing, just "who's home"). Polite enough to run on networks you're authorized on.

The third option, scanning a public IP, is where authorization matters most. Don't scan public IPs you don't own without explicit written permission. The default behavior is to scan only `nmap`'s own practice server, `scanme.nmap.org`, which they explicitly allow.

## Why every home network reuses the same numbers

You'll see `192.168.1.x` on basically every wifi network you connect to. That's not a coincidence or a security problem. Every router manufacturer picked the same default range because RFC 1918 made it free to do so, and because the network never leaves the building it doesn't matter if everyone uses the same numbers.

Some routers pick `192.168.0.x` (Linksys old defaults) or `192.168.50.x` (ASUS) or `10.0.0.x` (Xfinity) but the principle is the same. Pick a private range, hand out IPs from it, translate to public when traffic leaves.

## Quick reference card

```
Private IP ranges (RFC 1918):
  10.0.0.0/8
  172.16.0.0/12
  192.168.0.0/16

Find your network:
  ifconfig | grep "inet " | grep -v 127.0.0.1
  netstat -nr | grep default

Common CIDRs:
  /24 = 256 IPs (home)
  /16 = 65,536 IPs (company)
  /32 = 1 IP (single host)

Scan your network:
  sudo nmap -sn YOUR.NETWORK.RANGE.0/24

Public IP (handle with care):
  curl ifconfig.me
```

## Sources

- RFC 1918: Address Allocation for Private Internets
- Real recon sessions on home networks, May 2026
- Related: [`security/mac-audit.md`](./mac-audit.md), [`security/authorization-and-scope.md`](./authorization-and-scope.md)
