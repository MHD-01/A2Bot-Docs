# What You Need

## Hardware

If you're joining a workshop or class that already has built robots, you mainly need the laptop line below — see [Setup 2 — Laptop](../part3/setup-2-laptop.md). If you're building from scratch, see the full parts list in [Electronics & Hardware](../part2/electronics-hardware.md); the essentials are:

- A Raspberry Pi (4 recommended) with a fresh Ubuntu install, SD card, and a **5V/3A minimum** power supply — undervoltage causes flaky USB and WiFi behavior on a Pi (see [Troubleshooting Index](../appendices/troubleshooting-index.md)).
- An Arduino Nano (or clone) running the A2Bot motor-control firmware.
- The A2Bot chassis: motors, wheels, caster, an RPLidar, and a GY-85 IMU breakout.
- A laptop (Linux recommended — Ubuntu 22.04 matches the Pi's OS most closely and avoids version-mismatch surprises).

## Software

- **Ubuntu 22.04** on both the Pi and, ideally, your laptop. ROS 2 Humble targets this release.
- **ROS 2 Humble** — installed in [Part 1](../part1/install-and-source.md).
- A terminal you're comfortable in. Everything in this site is done from the command line — see [Ubuntu Terminal Basics](../toolkit/ubuntu-terminal-basics.md) if you're new to this.

## Accounts & access

- SSH access to the Pi (a username/password, or an SSH key) if you're joining an already-built robot — see [SSH & Remote Access](../toolkit/ssh-and-remote-access.md) if this is unfamiliar.
- No cloud accounts, ROS account, or sign-up is needed — everything here runs locally on your own machines.

## A note on pacing

Part 1 assumes **zero** prior ROS 2 knowledge and uses only a simulated turtle — no robot required yet. If you already know ROS 2 basics (nodes, topics, `colcon build`), skim it and move to Part 2.
