# SSH & Remote Access

## What this is, and why it matters

The robot's Raspberry Pi has no monitor or keyboard attached during normal use — every interaction with it happens remotely, from your laptop, over the network. **SSH** (Secure Shell) is what makes that possible: it opens a real terminal session on the Pi, running commands on it as if you were sitting in front of it, with everything encrypted over the connection. This page covers the underlying `ssh` command itself — a separate future Toolkit page will cover using SSH from inside an editor like VS Code specifically, which is a different (if related) workflow built on top of this one.

## The basic command

The shape is always the same: `ssh <username>@<host>`. In this course, that's the robot's own username and its `.local` hostname, both built from the robot's number (see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full naming convention):

!!! laptop "💻 Laptop"
    ```bash
    ssh a2botX@a2botX-host.local
    ```

Substitute the actual robot number for `X` in both places — a session on robot 5 would be `ssh a2bot5@a2bot5-host.local`. This is always run **from your laptop**, never on the Pi itself — you're using SSH to reach the Pi, not the other way around.

## The first-connection prompt: "authenticity of host"

The very first time you `ssh` into a given robot, you'll see something like this before it asks for a password:

```
The authenticity of host 'a2bot5-host.local (10.42.0.5)' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This isn't an error, and it isn't specific to this project — SSH shows it because it has no prior record of this particular machine, and is asking you to confirm you're actually connecting to the device you meant to. Type `yes` and press Enter to continue; SSH remembers this machine afterward (in `~/.ssh/known_hosts`) and won't ask again for that same hostname. You'll see this prompt again for each robot the first time you connect to it, since every robot is a different machine as far as SSH is concerned.

## When the hostname doesn't resolve

The `a2botX-host.local` form depends on mDNS working on whatever network you're currently on (see [IP Addresses & Your Network](ip-addresses-and-your-network.md) for why), which isn't guaranteed on every network. If `ssh a2botX@a2botX-host.local` can't find the host, fall back to connecting by the robot's raw IP address instead — both forms are shown side by side in [Setup 2, step 5](../part3/setup-2-laptop.md#5-get-on-the-same-network-as-the-robot) and the [Troubleshooting Index's Static Ethernet Link Setup entry](../appendices/troubleshooting-index.md#static-ethernet-link-setup). The dashboard's own post-connect panel (see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md#4-reconnect-and-get-ssh-access)) will show you the exact working command for whichever address currently applies — that's the authoritative source in the moment, over trying to reconstruct it yourself.

## Password auth — what this project actually uses

SSH supports both password-based login and key-based login (where a cryptographic key on your laptop proves your identity instead of typing a password each time). This course's robots use **password authentication**: every robot has a shared account password, shown directly on the dashboard's post-connect panel right alongside the `ssh` command itself, so it's never something you need to remember or look up separately. When you connect, you'll be prompted for that password after the host-authenticity check above.

Key-based login is a real alternative — [What You Need](../part0/what-you-need.md) mentions it as an option if you're joining a robot someone else already administers with keys set up — but it isn't this project's own default setup, so it isn't covered further here.

## Where this shows up in the course

You'll use this constantly once a robot is built or joined: [Discovering and Controlling A2Bot, step 4](../part0/discovering-and-controlling.md#4-reconnect-and-get-ssh-access) is the first real SSH session of the course, and [Setup 2, step 7](../part3/setup-2-laptop.md#7-ssh-in-optional) uses the same command any time you need a terminal on the Pi itself rather than just seeing its ROS 2 topics over the network.
