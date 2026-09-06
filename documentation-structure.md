# A2Bot Documentation — Structure

Built with MkDocs Material. 23 pages across 6 top-level sections.

## Part 0 — Getting Started

Intro, prerequisites, and the full first-contact flow: find the robot's hotspot, connect it to WiFi, SSH in, set up ROS 2, and drive it.

- What is A2Bot
- What You Need
- Choose Your Path
- Discovering and Controlling A2Bot

## Toolkit

Standalone reference pages for the general skills the workshop assumes — linked into from wherever they're needed, not read top-to-bottom.

- Ubuntu Terminal Basics
- IP Addresses & Your Network
- SSH & Remote Access
- VS Code Remote Development
- `~/.bashrc`
- Managing Multiple Terminals
- ROS 2 CLI Reference
- `rqt`
- `nmcli` & Network Connections
- `systemd` & Services

## Part 1 — ROS 2 Fundamentals

ROS 2 basics using the stock `turtlesim` before touching real hardware.

- Install & Source
- CLI Tools & Core Concepts
- Writing Your Own Nodes

## Part 2 — How A2Bot Is Built

The robot's software architecture, package by package.

- Electronics & Hardware
- Software Architecture
- The Driver: `a2bot_driver`
- The Robot Model: `a2bot_description`
- Sensor Fusion / EKF
- Closed-Loop Motion: `a2bot_service`

## Part 3 — Setup, Connection & Use

End-to-end operating instructions.

- Setup 1 — Raspberry Pi
- Setup 2 — Laptop
- ROS2 on Real Hardware
- Bringup & Driving
- Discovery & Dashboard
- SLAM & Navigation
- Capstone

## Appendices

- Flashing the Arduino Firmware
- Troubleshooting Index — symptom-first index of known gotchas
- Quick Reference — parameters, commands, addresses
