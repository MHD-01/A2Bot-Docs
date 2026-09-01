# IP Addresses & Your Network

## What this is, and why it matters

Your laptop and the robot's Pi are two separate computers, and for ROS 2 to let them work as one system, they first have to be able to reach each other at all — a much more basic requirement than anything ROS 2-specific. This page covers just enough about IP addresses and networks to understand *why* a setup step asks you to check an address or join a particular network, without turning into a full networking course.

## What an IP address actually is

An IP address is just a number identifying one device on a network — something like `10.42.0.1`. Two devices can only talk to each other directly if they're on the **same network**: plugged into the same router, joined to the same WiFi, or connected by a direct cable. Being able to see a network in a WiFi list, or having "internet access," doesn't by itself mean two specific devices can reach each other — that's exactly the situation [Setup 2, step 5](../part3/setup-2-laptop.md#5-get-on-the-same-network-as-the-robot) walks through getting right.

## Subnets: enough to reason about "same network"

An address is normally written with a subnet size attached, like `10.0.0.5/24`. The `/24` says the first three numbers (`10.0.0`) identify the *network*, and only the last number (`5`) identifies *this device* on it — every other device on that same network also starts with `10.0.0.` and differs only in that last number.

This is why `192.168.0.x` and `192.168.1.x` are **different** networks even though they look almost identical — the third number is part of the network identity, not just a device number. Two devices on those two different `/24`s can't reach each other directly, no router or configuration change aside.

This project's direct-Ethernet link (see [Direct Ethernet link: a fixed private network](#direct-ethernet-link-a-fixed-private-network)) is a concrete `/24` example: the Pi takes `10.0.0.10X` (X being the robot's own number, so robot 5 is `10.0.0.105`) and the laptop takes a fixed `10.0.0.250` — both on the same `10.0.0.0/24` network, which is exactly what lets them talk directly over the cable with no router in between.

## Direct Ethernet link: a fixed private network

A direct Ethernet cable between the laptop and the Pi sidesteps WiFi entirely — no router, no AP/client isolation, no signal strength issues. This is a good learning example because it makes the network rules simple: both devices get addresses on the same private `/24` network, and each end knows exactly where the other one is.

**The plan:** no DHCP exists on a direct cable, so both ends get fixed addresses on a private `10.0.0.0/24` subnet. Do not set a gateway on this link — a gateway here can make the Pi try to route internet traffic down the dead-end cable instead of over WiFi.

- **Pi:** `10.0.0.10X` — the last octet is **100 plus** the robot's own unique number (matching its hostname, hotspot, and `ROS_DOMAIN_ID` — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md)). Robot 5 is `10.0.0.105`.
- **Laptop:** `10.0.0.250` — a fixed address reserved for this purpose, chosen to stay outside the `10.0.0.100`–`10.0.0.201` range robot numbers occupy, so it never collides with any robot's own address.

!!! pi "🤖 Pi"
    ```bash
    sudo nmcli connection add type ethernet con-name eth0-static ifname eth0 \
      ipv4.method manual ipv4.addresses 10.0.0.10X/24 \
      connection.autoconnect yes
    sudo nmcli connection up eth0-static
    ```

    Substitute `100 + the robot's number` for `10X` (robot 5 → `10.0.0.105`). On a fresh Ubuntu Server Pi, netplan may hand `eth0` to `systemd-networkd` instead of NetworkManager, in which case this profile will not survive a reboot. Check with `cat /etc/netplan/*.yaml` — if it does not say `renderer: NetworkManager`, either add that line and `sudo netplan apply`, or configure the static IP natively in a netplan YAML file instead of via `nmcli`.

Linux names interfaces by hardware type and location, not by a fixed `eth0`/`wlan0` scheme anymore — Ethernet ports show up as `enp3s0`, `eno1`, or (for a USB-to-Ethernet dongle) `enx<mac-address>`, while WiFi radios show up as `wlp2s0` or `wlan0`. The prefix is the signal: `en` means Ethernet, `wl` means wireless. You want an `en*` name, never a `wl*` one.

To find the exact name:

!!! laptop "💻 Laptop"
    ```bash
    nmcli device status
    ```

    Look for the row with `TYPE` set to `ethernet`. Before the cable is plugged in it typically shows `unavailable`; plug the cable into both the laptop and the Pi and run the command again — that row should flip to `connecting` or `disconnected` (the "link detected, no IP yet" state), confirming it is the right interface.

    If more than one `ethernet` row appears (for example, a built-in port plus a USB dongle), unplug the cable, run the command again, then plug it back in and compare — whichever row's state changes is your interface. `ip a` (or `ip -o link show`) shows the same information in a more raw format if you prefer it.

    If this laptop also uses its Ethernet port on other networks, set `connection.autoconnect no` on this profile so it does not grab the interface unexpectedly elsewhere.

!!! laptop "💻 Laptop"
    ```bash
    sudo nmcli connection add type ethernet con-name robot-link ifname <your-iface> \
      ipv4.method manual ipv4.addresses 10.0.0.250/24 \
      connection.autoconnect yes
    sudo nmcli connection up robot-link
    ```

    Substitute the interface name you found above for `<your-iface>`.

**Test:**

!!! laptop "💻 Laptop"
    ```bash
    ping 10.0.0.10X
    ssh a2botX@a2botX-host.local
    ```

    Substitute the robot's actual number for `X` (robot 5 → `ping 10.0.0.105`). If `.local` mDNS resolution does not work, `ssh a2botX@10.0.0.10X` (by IP) works over this link regardless.

### Static IP works but no internet

**Symptom:** After setting up the Ethernet link, the Pi can reach the laptop but `apt install` fails with `Temporary failure resolving ...`.

**Cause:** Something is stealing the default route away from WiFi — either the Ethernet profile has a gateway set (it should not), or the WiFi radio is in access-point mode instead of client mode (a single radio can only be one or the other).

**Diagnose:**

!!! pi "🤖 Pi"
    ```bash
    ip route              # look for "default via ... dev wlan0"
    nmcli connection show --active     # is an AP profile (e.g. A2Bot-AP) holding wlan0?
    ```

**Fix:** disable the AP profile's autoconnect and bring up a real WiFi client connection:

!!! pi "🤖 Pi"
    ```bash
    nmcli connection modify A2Bot-AP connection.autoconnect no
    nmcli connection down A2Bot-AP
    nmcli device wifi connect "YOUR_SSID" password "YOUR_PASSWORD"
    ```

    The direct Ethernet link keeps working in both modes regardless — it is a separate interface, so you can always reach `10.0.0.10X` over the cable even while the Pi has no WiFi internet.

## Private vs. public IP ranges, briefly

Address ranges starting with `10.x.x.x`, `172.16.x.x`–`172.31.x.x`, or `192.168.x.x` are **private** — reserved for local networks and never reachable directly from the open internet. Every address you'll actually see in this course is one of these: the robot's hotspot lives at `10.42.0.X`, and the direct-Ethernet link uses `10.0.0.x` — both private, both meaningful only on that one specific local link, never as an address you could reach from anywhere else.

## Hostnames and `.local` resolution (mDNS)

Typing a raw IP address works, but it's easy to mistype and can change between sessions. **mDNS** lets devices on the same local network resolve a friendly `<name>.local` hostname to whatever IP that device currently has — no internet connection or DNS server required, just both devices being on the same network.

This is exactly what this course's SSH flow relies on: connecting with `a2botX-host.local` (X being the robot's number) rather than remembering its numeric address, as [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md#4-reconnect-and-get-ssh-access) and [Setup 2](../part3/setup-2-laptop.md) both use. mDNS isn't universally supported on every network or OS, though — when `.local` resolution fails, falling back to the robot's raw IP address is the documented recovery, covered in [Setup 2, step 5](../part3/setup-2-laptop.md#5-get-on-the-same-network-as-the-robot) and the [Direct Ethernet link: a fixed private network](#direct-ethernet-link-a-fixed-private-network) section above.

## This project's real addresses, gathered in one place

| What | Address | Notes |
|---|---|---|
| Robot's setup hotspot | `10.42.0.X` | X = the robot's own number — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) |
| Direct-Ethernet link, Pi side | `10.0.0.10X` | 100 + the robot's own number |
| Direct-Ethernet link, laptop side | `10.0.0.250` | Fixed, chosen to stay outside the `.100`–`.201` range robot numbers occupy |
| Hostname (any connection method) | `a2botX-host.local` | X = the robot's own number |

## A different kind of "address": `ROS_DOMAIN_ID`

You'll also see every robot assigned a `ROS_DOMAIN_ID` — don't confuse this with an IP address. It isn't a networking-layer concept at all; it's a ROS 2-level number that determines which ROS 2 processes are willing to discover each other, even among machines that are already on the same IP network and can already `ping`/`ssh` each other fine. Two machines can be perfectly reachable at the IP level and still see none of each other's ROS 2 topics if this number doesn't match. The full mechanics belong to ROS 2 itself, not general networking — covered where the course actually sets it (see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md)), not on this page.

## Where this shows up in the course

Every "get on the same network as the robot" step in [Setup 2](../part3/setup-2-laptop.md) and [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) assumes this page's mental model. If `ping` and `ssh` both work but ROS 2 still can't see the robot's topics, that's not an IP-addressing problem at all — see [Troubleshooting Index — ROS 2 topics work over ping/SSH but discovery fails](../appendices/troubleshooting-index.md#ros-2-topics-work-over-pingssh-but-discovery-fails).
