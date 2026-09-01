# Setup 2 — Laptop

This page gets your laptop able to see and drive A2Bot. If you just finished [Setup 1 — Raspberry Pi](setup-1-raspberry-pi.md), continue here to connect to the robot you built. If a robot is already built and running — a classroom or workshop robot someone else set up — this is the **only** setup page you need; you don't need to touch the Raspberry Pi at all.

## 1. Install ROS 2 Humble on your laptop

!!! laptop "💻 Laptop"
    Follow [Install & Source](../part1/install-and-source.md) if you haven't already.

## 2. Install and configure Cyclone DDS

ROS2 can run on several different DDS implementations underneath, and two machines must use the *same* one to discover each other reliably. Cyclone DDS is used here because it is what the robots themselves are already configured with — a mismatched RMW implementation between a laptop and a robot is a real, silent cause of "why can't I see any topics" that matches no error message.

!!! laptop "💻 Laptop"
    Install:
    ```bash
    sudo apt install ros-humble-rmw-cyclonedds-cpp
    ```

    Set as the active RMW (added to `~/.bashrc` so it persists):
    ```bash
    echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
    source ~/.bashrc
    ```

    Verify it's active:
    ```bash
    echo $RMW_IMPLEMENTATION
    ros2 pkg list | grep cyclonedds
    ```

!!! warning "Must match the robot exactly"
    This has to be installed and set **identically** on both machines, or discovery between them can fail even when `ROS_DOMAIN_ID` matches. If you built the Pi yourself in Setup 1, you already did this there; if you're joining an existing robot, it's already done on the Pi's side.

## 3. Build the workspace locally

Even though the robot's own workspace is already built on its Pi, RViz on your laptop needs its **own local build** — it resolves the URDF's mesh files (`package://a2bot_description/meshes/...`) from the local filesystem via the package index, not over the network. A laptop that's never built `a2bot_description` shows an outline-only or invisible robot in RViz, even while topics and tf work fine remotely. See [Troubleshooting Index](../appendices/troubleshooting-index.md#rviz-shows-no-robot-model) if this happens.

!!! laptop "💻 Laptop"
    ```bash
    git clone <your-a2bot-repo-url> ~/a2bot
    cd ~/a2bot/a2bot_ws
    rosdep install --from-paths src --ignore-src -r -y
    colcon build --symlink-install
    echo "source ~/a2bot/a2bot_ws/install/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```

## 4. Match the ROS Domain ID

Your `ROS_DOMAIN_ID` must match the robot's exactly, not the ROS 2 default of 0. Which value that is depends on how you got here:

- **You set up the Pi yourself in Setup 1** — use whatever `ROS_DOMAIN_ID` you set there.
- **You're joining an existing, already-built robot** — its domain ID is the robot's own unique number, the same one baked into its hostname and hotspot name (e.g. `a2bot5-host` → domain ID `5`). See [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full naming convention.
- **Still not sure?** SSH into the robot ([step 7](#7-ssh-in-optional)) and check directly:
  ```bash
  echo $ROS_DOMAIN_ID
  ```

!!! laptop "💻 Laptop"
    ```bash
    echo "export ROS_DOMAIN_ID=<robot's number>" >> ~/.bashrc
    source ~/.bashrc
    ```

    It must match exactly on every machine that needs to see the robot.

## 5. Get on the same network as the robot

For ROS 2 discovery to work, both machines need to reach each other. Three supported ways to get there:

- **A direct Ethernet cable** — the most reliable option, with no WiFi variables involved. The robot's Pi takes a fixed address of `10.0.0.X` (its own robot number), and your laptop needs a matching static address on the same `10.0.0.0/24` block — `10.0.0.200` is the fixed convention used here, chosen so it never collides with a robot number. Full `nmcli`/`netplan` setup for both ends is documented in [Troubleshooting Index](../appendices/troubleshooting-index.md#static-ethernet-link-setup).
- **Your own personal hotspot** — if you're joining an already-built robot and want to take it off the classroom network, connect the robot to your phone's hotspot by following the discovery sequence in [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) (Part 0) — it walks through the robot's own dashboard flow for this. Once the robot is on your hotspot and your laptop is too, you're on the same network.
- **Shared WiFi** — simplest when a suitable network is already available; just join both machines to it.

!!! laptop "💻 Laptop"
    Make sure an SSH client is installed — most Linux distros ship one by default, but if `ssh` isn't found:
    ```bash
    sudo apt install openssh-client
    ```

Once both machines are on the same network, connect using the robot's hostname (matching its number, from [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md)):

```bash
ssh a2botX@a2botX-host.local
```

If `.local` mDNS resolution doesn't work on your network, fall back to connecting by IP — see [step 7](#7-ssh-in-optional).

## 6. Verify you can see the robot

With the robot's driver chain already running (from [Setup 1](setup-1-raspberry-pi.md) if you built it, or someone else's job if you're joining):

!!! laptop "💻 Laptop"
    ```bash
    ros2 topic list
    ros2 topic hz /odom
    ```

If topics like `/odom`, `/scan`, and `/joint_states` show up and `/odom` is actually publishing, you're connected.

## 7. SSH in (optional)

If you also need a terminal on the Pi itself:

!!! laptop "💻 Laptop"
    ```bash
    ssh <username>@<robot-ip>
    ```

Ask the robot's owner for the username and IP — see [Discovery & Dashboard](discovery-and-dashboard.md) if the robot exposes its dashboard, which often shows the current IP.

## 8. Install VS Code and Terminator (optional)

Neither is required for anything above — both are productivity tools for what comes next, not a dependency of ROS 2 or A2Bot itself.

!!! laptop "💻 Laptop"
    ```bash
    sudo apt install code
    ```

    Then, inside VS Code, install the **Remote-SSH** extension from the Extensions panel. See [VS Code Remote Development](../toolkit/vscode-remote-development.md) for how to connect to the robot through it once installed.

    ```bash
    sudo apt install terminator
    ```

    A GUI terminal that splits into multiple panes — see [Managing Multiple Terminals](../toolkit/managing-multiple-terminals.md) for how to use it.

Next: [ROS2 on Real Hardware](ros2-on-real-hardware.md).
