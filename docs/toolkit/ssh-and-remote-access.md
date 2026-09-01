# SSH & Remote Access

## What this is, and why it matters

The robot's Raspberry Pi has no monitor or keyboard attached during normal use — every interaction with it happens remotely, from your laptop, over the network. **SSH** (Secure Shell) is what makes that possible: it opens a real terminal session on the Pi, running commands on it as if you were sitting in front of it, with everything encrypted over the connection. This page covers the underlying `ssh` command itself — a separate future Toolkit page will cover using SSH from inside an editor like VS Code specifically, which is a different (if related) workflow built on top of this one.

## Install SSH on your laptop first

Before you try to connect, make sure the SSH client is installed on the machine you are using as your laptop or development computer.

On Ubuntu or Debian-based Linux:

```bash
sudo apt update
sudo apt install openssh-client
```

On macOS, OpenSSH is usually already installed as part of the base system. You can check with:

```bash
ssh -V
```

On Windows, use the built-in OpenSSH client if available, or install the Windows OpenSSH package through the system settings or PowerShell. If `ssh` is not recognized, install the OpenSSH client and reopen the terminal before continuing.

Once the command is available, confirm it works by running:

```bash
ssh -V
```

If that prints a version string, you're ready to connect.

## Make sure the robot and your device are on the same network

You can only reach the robot over SSH when your laptop and the robot are on the same local network. This is a required first check before you even try to connect.

### WiFi case

If the robot is connected by WiFi, your laptop must also be connected to the same WiFi network as the robot. In other words, both devices need to be on the same LAN. If the robot is on one WiFi network and your laptop is on another, the host will not resolve and the SSH session will fail.

### Ethernet case

If the robot is connected by Ethernet, your laptop should also be connected to the same local network, usually through the same router, switch, or direct network setup. When using a wired connection, the robot and your laptop still must share the same network segment for SSH to work reliably.

If you are unsure whether both devices are on the same network, check the robot's IP address and compare it to the address range of the network your laptop is using. A mismatch here is the most common reason SSH appears to "not work".

## QR code information for the robot

The robot dashboard or setup display includes a QR code that holds the robot connection information. This is meant to help you connect quickly without having to remember the IP address or password. Scan the QR code to see the robot's current static IP and the SSH password for that robot.

This is especially useful when you are first setting up a robot or reconnecting after a reboot. The QR code gives you the information needed to start the SSH session with the correct address and password.

## The basic command

The shape is always the same: `ssh <username>@<host>`. In this course, that's the robot's own username and its `.local` hostname, both built from the robot's number (see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full naming convention):

!!! laptop "💻 Laptop"
    ```bash
    ssh a2botX@a2botX-host.local
    ```

Substitute the actual robot number for `X` in both places — a session on robot 5 would be `ssh a2bot5@a2bot5-host.local`. This is always run **from your laptop**, never on the Pi itself — you're using SSH to reach the Pi, not the other way around.

## Confirm the connection before you continue

Before the SSH session is established, your laptop checks whether the host is the expected device. This is a normal part of SSH and is the confirmation step that prevents connecting to the wrong machine.

The very first time you `ssh` into a given robot, you'll see something like this before it asks for a password:

```
The authenticity of host 'a2bot5-host.local (10.42.0.5)' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is not an error. SSH is telling you that it has no stored record of that host yet and is asking you to confirm that the device you are connecting to is the robot you meant to reach. Type `yes` and press Enter to continue. After that, SSH remembers this machine and you usually won't see this prompt again for that same robot.

If you are not on the same network, or the connection is going to a different machine, the host may not resolve correctly or the connection may fail. In that case, stop and check the network setup before continuing.

## When the hostname doesn't resolve

The `a2botX-host.local` form depends on mDNS working on whatever network you're currently on (see [IP Addresses & Your Network](ip-addresses-and-your-network.md) for why), which isn't guaranteed on every network. If `ssh a2botX@a2botX-host.local` can't find the host, fall back to connecting by the robot's raw IP address instead — both forms are shown side by side in [Setup 2, step 5](../part3/setup-2-laptop.md#5-get-on-the-same-network-as-the-robot) and the [Troubleshooting Index's Static Ethernet Link Setup entry](../appendices/troubleshooting-index.md#static-ethernet-link-setup). The dashboard's own post-connect panel (see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md#4-reconnect-and-get-ssh-access)) will show you the exact working command for whichever address currently applies — that's the authoritative source in the moment, over trying to reconstruct it yourself.

## Password auth — what this project actually uses

SSH supports both password-based login and key-based login, and both are valid ways to authenticate in general. This course's robots use **password authentication** as the standard setup: every robot has a shared account password, shown directly on the dashboard's post-connect panel right alongside the `ssh` command itself, so it's never something you need to remember or look up separately. When you connect, you'll be prompted for that password after the host-authenticity check above.

SSH key authentication also exists as a supported option in many environments, but it is not the default workflow for this project and is not described here in detail. The important point is that you do not need to set up any special key infrastructure just to start using the robot in this course.

## Where this shows up in the course

You'll use this constantly once a robot is built or joined: [Discovering and Controlling A2Bot, step 4](../part0/discovering-and-controlling.md#4-reconnect-and-get-ssh-access) is the first real SSH session of the course, and [Setup 2, step 7](../part3/setup-2-laptop.md#7-ssh-in-optional) uses the same command any time you need a terminal on the Pi itself rather than just seeing its ROS 2 topics over the network.
