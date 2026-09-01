# IP Addresses & Your Network

## What this is, and why it matters

Your laptop and the robot's Pi are two separate computers, and for ROS 2 to let them work as one system, they first have to be able to reach each other at all — a much more basic requirement than anything ROS 2-specific. This page covers just enough about IP addresses and networks to understand *why* a setup step asks you to check an address or join a particular network, without turning into a full networking course.

## What an IP address actually is

An IP address is just a number identifying one device on a network — something like `10.42.0.1`. Two devices can only talk to each other directly if they're on the **same network**: plugged into the same router, joined to the same WiFi, or connected by a direct cable. Being able to see a network in a WiFi list, or having "internet access," doesn't by itself mean two specific devices can reach each other — that's exactly the situation [Setup 2, step 5](../part3/setup-2-laptop.md#5-get-on-the-same-network-as-the-robot) walks through getting right.

## Subnets: enough to reason about "same network"

An address is normally written with a subnet size attached, like `10.0.0.5/24`. The `/24` says the first three numbers (`10.0.0`) identify the *network*, and only the last number (`5`) identifies *this device* on it — every other device on that same network also starts with `10.0.0.` and differs only in that last number.

This is why `192.168.0.x` and `192.168.1.x` are **different** networks even though they look almost identical — the third number is part of the network identity, not just a device number. Two devices on those two different `/24`s can't reach each other directly, no router or configuration change aside.

This project's direct-Ethernet link (see [Troubleshooting Index — Static Ethernet Link Setup](../appendices/troubleshooting-index.md#static-ethernet-link-setup)) is a concrete `/24` example: the Pi takes `10.0.0.X` (X being the robot's own number) and the laptop takes a fixed `10.0.0.200` — both on the same `10.0.0.0/24` network, which is exactly what lets them talk directly over the cable with no router in between.

## Private vs. public IP ranges, briefly

Address ranges starting with `10.x.x.x`, `172.16.x.x`–`172.31.x.x`, or `192.168.x.x` are **private** — reserved for local networks and never reachable directly from the open internet. Every address you'll actually see in this course is one of these: the robot's hotspot lives at `10.42.0.1`, and the direct-Ethernet link uses `10.0.0.x` — both private, both meaningful only on that one specific local link, never as an address you could reach from anywhere else.

## Hostnames and `.local` resolution (mDNS)

Typing a raw IP address works, but it's easy to mistype and can change between sessions. **mDNS** lets devices on the same local network resolve a friendly `<name>.local` hostname to whatever IP that device currently has — no internet connection or DNS server required, just both devices being on the same network.

This is exactly what this course's SSH flow relies on: connecting with `a2botX-host.local` (X being the robot's number) rather than remembering its numeric address, as [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md#4-reconnect-and-get-ssh-access) and [Setup 2](../part3/setup-2-laptop.md) both use. mDNS isn't universally supported on every network or OS, though — when `.local` resolution fails, falling back to the robot's raw IP address is the documented recovery, covered in [Setup 2, step 5](../part3/setup-2-laptop.md#5-get-on-the-same-network-as-the-robot) and the [Static Ethernet Link Setup](../appendices/troubleshooting-index.md#static-ethernet-link-setup) entry.

## This project's real addresses, gathered in one place

| What | Address | Notes |
|---|---|---|
| Robot's setup hotspot | `10.42.0.X` | X = the robot's own number — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) |
| Direct-Ethernet link, Pi side | `10.0.0.X` | X = the robot's own number |
| Direct-Ethernet link, laptop side | `10.0.0.200` | Fixed, chosen to never collide with a robot number |
| Hostname (any connection method) | `a2botX-host.local` | X = the robot's own number |

## A different kind of "address": `ROS_DOMAIN_ID`

You'll also see every robot assigned a `ROS_DOMAIN_ID` — don't confuse this with an IP address. It isn't a networking-layer concept at all; it's a ROS 2-level number that determines which ROS 2 processes are willing to discover each other, even among machines that are already on the same IP network and can already `ping`/`ssh` each other fine. Two machines can be perfectly reachable at the IP level and still see none of each other's ROS 2 topics if this number doesn't match. The full mechanics belong to ROS 2 itself, not general networking — covered where the course actually sets it (see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md)), not on this page.

## Where this shows up in the course

Every "get on the same network as the robot" step in [Setup 2](../part3/setup-2-laptop.md) and [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) assumes this page's mental model. If `ping` and `ssh` both work but ROS 2 still can't see the robot's topics, that's not an IP-addressing problem at all — see [Troubleshooting Index — ROS 2 topics work over ping/SSH but discovery fails](../appendices/troubleshooting-index.md#ros-2-topics-work-over-pingssh-but-discovery-fails).
