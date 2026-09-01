# Setup 1 — Raspberry Pi

!!! important "Is the robot already built and running?"
    If yes, **skip Setup 1 entirely** and go straight to
    [Setup 2 — Laptop](setup-2-laptop.md) — you do not need to touch the
    Raspberry Pi at all. Setup 1 is only for building a robot **from
    scratch**: a bare Raspberry Pi and bare chassis with nothing
    assembled or configured yet.

This page takes a bare Raspberry Pi and bare chassis to a driving, networked robot, in the order it actually needs to happen. Each step links to the page with full concept/detail rather than repeating it here.

## 1. Flash the Pi

Raspberry Pi Imager can write more than just the OS image — its settings menu pre-configures the image itself before it's ever booted, so the Pi comes up already join-able over the network with SSH ready. That's what makes the rest of this page possible with no monitor or keyboard ever connected to the Pi.

Flash **Ubuntu Server 22.04.5 LTS (64-bit)** — not Desktop, headless is all the Pi needs (matches [What You Need](../part0/what-you-need.md)). Before writing, open Imager's settings (the gear icon, or Ctrl+Shift+X) and set:

- **Hostname**: `a2botX` — `X` is this robot's own number, the same one that ties its hostname, hotspot, and identity together; see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full convention.
- **Username / password**: `a2botX` and a password of your choice — this is the account the rest of this page (and every future SSH session) logs into.
- **WiFi SSID / password**: only if a known network is available right now. If not, skip this — [Discovery & Dashboard](discovery-and-dashboard.md) covers recovering a Pi with no known network via its own fallback hotspot.
- **Enable SSH**: turns on the SSH server itself, so a remote login works starting from the very first boot.
- **Public key** (optional): paste your laptop's own SSH public key here for passwordless login into this specific Pi while you build it.

  !!! laptop "💻 Laptop"
      ```bash
      cat ~/.ssh/id_ed25519.pub
      ```
      No key yet? `ssh-keygen -t ed25519` creates one. This is a convenience for your own build session — it doesn't replace the shared account password other students will use to reach this robot afterward; see [SSH & Remote Access](../toolkit/ssh-and-remote-access.md) for the password-based workflow the rest of this course assumes.

Write the image, then boot the Pi with it connected to Ethernet or in range of the WiFi you configured above.

## 2. Wire the electronics

Assemble the chassis, wire the motors to the Arduino, and connect the Arduino, RPLidar, GY-85 IMU, and camera to the Pi's USB/I2C. See [Electronics & Hardware](../part2/electronics-hardware.md) for the full component list and pinout notes.

## 3. Update the system

A freshly flashed image's package list is already out of date, and everything installed later on this page should build on current versions, not stale ones from the image.

!!! pi "🤖 Pi"
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
    `update` refreshes the local list of what's available from each repository; `upgrade` actually installs newer versions of what's already on the system. This can take a while — a fresh image typically has a lot waiting, and speed depends entirely on the Pi's internet connection.

## 4. Disable automatic updates

Ubuntu Server ships with `unattended-upgrades` running on a timer in the background, applying its own updates whenever it feels like it. It holds the same apt lock every `apt install` on the rest of this page needs — if it's mid-run when you type one, that command just hangs waiting on a lock that might not free up for a while. Turning it off hands control of *when* updates happen back to you, which matters with the number of `apt install`s still ahead.

!!! pi "🤖 Pi"
    ```bash
    sudo systemctl disable --now unattended-upgrades apt-daily.timer apt-daily-upgrade.timer
    ```
    `--now` stops all three immediately, as well as preventing them from starting again at every future boot.

## 5. Install ROS 2 Humble

ROS 2 doesn't come from Ubuntu's own package repositories — it needs its own apt source registered first. The Pi uses the current officially-recommended method, a small `.deb` that adds ROS 2's repository and signing key in one step (a newer approach than the manual keyring method [Install & Source](../part1/install-and-source.md) walks through for the laptop; both end up with the same working install).

!!! pi "🤖 Pi"
    ```bash
    sudo apt install -y software-properties-common
    sudo add-apt-repository universe

    sudo apt update && sudo apt install -y curl
    export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
    curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo $VERSION_CODENAME)_all.deb"
    sudo apt install -y /tmp/ros2-apt-source.deb

    sudo apt update && sudo apt install -y ros-humble-desktop ros-dev-tools
    ```
    `ros-humble-desktop` is the same full install [Install & Source](../part1/install-and-source.md) uses on the laptop (RViz and demos included, even though the Pi itself won't run RViz); `ros-dev-tools` adds `colcon`, `rosdep`, and the rest of the build tooling the later steps on this page need.

Source it automatically in every new terminal, the same way every machine in this course does:

!!! pi "🤖 Pi"
    ```bash
    echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```
    See [~/.bashrc](../toolkit/bashrc.md) for why this pattern is used everywhere in this course instead of sourcing by hand each time.

## 6. Switch to Cyclone DDS

ROS 2 can run on several different DDS implementations underneath, and two machines must use the *same* one to discover each other reliably. Cyclone DDS is used here because it is what the robots themselves are already configured with — a mismatched RMW implementation between a laptop and a robot is a real, silent cause of "why can't I see any topics" that matches no error message.

!!! pi "🤖 Pi"
    ```bash
    sudo apt install -y ros-humble-rmw-cyclonedds-cpp
    echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
    source ~/.bashrc
    ```

    Verify it's active:
    ```bash
    echo $RMW_IMPLEMENTATION
    ros2 pkg list | grep cyclonedds
    ```

## 7. Set the ROS Domain ID

`ROS_DOMAIN_ID` is a ROS 2-level number, not a network address — it decides which ROS 2 processes are even willing to discover each other, independent of whether they can already `ping`/`ssh` each other fine. See [IP Addresses & Your Network — A different kind of "address"](../toolkit/ip-addresses-and-your-network.md#a-different-kind-of-address-ros_domain_id) if you want the fuller mental model.

!!! pi "🤖 Pi"
    ```bash
    echo "export ROS_DOMAIN_ID=X" >> ~/.bashrc
    source ~/.bashrc
    ```

    Substitute this robot's own number for `X` (robot 5 → `ROS_DOMAIN_ID=5`) — **not** the ROS 2 default of 0, and **not** shared with any other robot in the classroom. It's the same `X` that names this robot's hostname and hotspot; see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full convention. It must match exactly on every machine that needs to see the robot — see [Setup 2 — Laptop](setup-2-laptop.md#4-match-the-ros-domain-id) for setting it on your laptop to match.

## 8. Install ROS 2 robot dependencies

Beyond the base `ros-humble-desktop` install, A2Bot's own packages depend on several more ROS 2 packages:

| Package | Purpose |
|---|---|
| `ros-humble-navigation2` | Nav2 itself — autonomous path planning and obstacle avoidance |
| `ros-humble-nav2-bringup` | Launch files and default config that tie Nav2's many nodes together |
| `ros-humble-slam-toolbox` | Builds a map from lidar scans while driving (SLAM) |
| `ros-humble-robot-localization` | The EKF that fuses wheel odometry and IMU data into one pose estimate — see [Sensor Fusion / EKF](../part2/sensor-fusion-ekf.md) |
| `ros-humble-rplidar-ros` | Driver for the RPLidar A1 |
| `ros-humble-camera-ros` | Driver for the Pi camera |
| `ros-humble-teleop-twist-keyboard` | Manual keyboard driving, for testing before autonomy is involved |
| `ros-humble-rosbridge-suite` | WebSocket bridge that lets a browser (the dashboard, the web control interface) talk to ROS 2 topics |
| `ros-humble-tf2-tools` | `tf2` inspection/debugging utilities |

You don't have to install these one by one — every package here is already declared as a dependency in A2Bot's own `package.xml` files, so once the workspace is cloned (next), `rosdep` installs all of them automatically:

!!! pi "🤖 Pi"
    ```bash
    cd ~/a2bot/a2bot_ws
    rosdep install --from-paths src --ignore-src -r -y
    ```

    The table above exists so you know *what* just got installed and *why* — see [step 12](#12-clone-and-build-the-a2bot-workspace) below for where this actually runs in sequence.

## 9. Hardware enablement

### Camera overlay

A CSI ribbon camera needs its sensor driver enabled in the Pi's firmware config before Linux will see it — a USB camera needs none of this and can be skipped straight to the next section.

!!! pi "🤖 Pi"
    ```bash
    sudo nano /boot/firmware/config.txt
    ```
    Add a line naming your specific sensor, for example:
    ```
    dtoverlay=imx219
    ```
    Substitute the overlay for whatever sensor your camera module actually uses — check the module's documentation if you're not sure. Reboot for it to take effect.

### Stable device names (udev rules)

Linux assigns `/dev/ttyUSB0`-style names by **plug-in order**, not by device identity — plug the Arduino in after the lidar one day and the names swap. A2Bot's code never references a raw `/dev/ttyUSB*` name; it always uses fixed symlinks created by udev rules that match each device's USB vendor/product ID instead of plug-in order.

!!! pi "🤖 Pi"
    ```bash
    sudo tee /etc/udev/rules.d/99-a2bot.rules > /dev/null <<'EOF'
    SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", SYMLINK+="arduino"
    SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", SYMLINK+="rplidar"
    EOF
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    ```
    `1a86:7523` is the Arduino's CH340 USB-serial chip; `10c4:ea60` is the RPLidar's CP210x chip. Confirm both symlinks exist after replugging the devices: `ls -l /dev/arduino /dev/rplidar`.

### Remove brltty

Ubuntu's `brltty` service (accessibility support for braille displays) misidentifies the Arduino's CH340 chip as braille hardware and silently claims the port a few seconds after it appears — `/dev/arduino` shows up, then vanishes, with no error. Removing it up front avoids ever hitting this:

!!! pi "🤖 Pi"
    ```bash
    sudo apt remove -y brltty
    ```

!!! danger "Already hitting this instead?"
    If `/dev/arduino` is appearing and disappearing, see [Troubleshooting Index — brltty steals the Arduino](../appendices/troubleshooting-index.md#brltty-steals-the-arduino) for full diagnosis — this is a near-universal first-build snag on Ubuntu.

### Enable I2C

The GY-85 IMU talks to the Pi over I2C, which is disabled by default. Unlike Raspberry Pi OS, Ubuntu Server has no `raspi-config` — I2C is enabled the same way as the camera, via the firmware config file:

!!! pi "🤖 Pi"
    ```bash
    sudo nano /boot/firmware/config.txt
    ```
    Add:
    ```
    dtparam=i2c_arm=on
    ```
    Then install the tools used to talk to I2C devices directly (used throughout [Troubleshooting Index](../appendices/troubleshooting-index.md#the-gy-85-reads-plausible-garbage-instead-of-erroring) to check the IMU is alive):
    ```bash
    sudo apt install -y i2c-tools
    ```
    Reboot for the config change to take effect, then confirm both IMU chips answer:
    ```bash
    sudo i2cdetect -y 1
    ```
    See [Electronics & Hardware — The GY-85 is three chips, not one](../part2/electronics-hardware.md#the-gy-85-is-three-chips-not-one) for what you should see at which address.

### Set required Pi groups

Each group below lets a normal user account (rather than `sudo` every time) access one piece of hardware:

| Group | Grants access to |
|---|---|
| `dialout` | Serial ports — the Arduino and RPLidar |
| `i2c` | The IMU |
| `video` | The camera |
| `netdev` | WiFi management via `nmcli` |

!!! pi "🤖 Pi"
    ```bash
    sudo usermod -aG dialout,i2c,video,netdev $USER
    ```

    Log out and back in (or reboot) — group membership doesn't apply to an already-open session.

## 10. Python and system packages

!!! pi "🤖 Pi"
    ```bash
    pip install pyserial
    ```
    `serial_bridge` (see [The Driver: a2bot_driver](../part2/a2bot-driver.md#serial_bridge)) uses this to talk to the Arduino over `/dev/arduino`.

Node.js and npm are needed on the Pi for the web control interface (`a2bot-webui`, see [step 13](#13-set-up-the-systemd-services)) — Ubuntu 22.04's own apt repository ships a Node version too old for it, so install from NodeSource instead:

!!! pi "🤖 Pi"
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
    sudo apt install -y nodejs
    ```
    Confirm with `node -v` (18 or newer). Then, inside the web control interface's own directory in the workspace (check `a2bot-webui.service`'s `WorkingDirectory=` if you're not sure which one):
    ```bash
    npm install
    ```

!!! note "MediaPipe / gesture control does not go on the Pi"
    Gesture control runs on your **laptop**, not the Pi — it needs a webcam and more spare compute than a Pi 4 has left over while also running SLAM and Nav2. See [Discovery & Dashboard — Gesture control](discovery-and-dashboard.md#gesture-control-optional-demo) for the install step, which belongs on Setup 2, not here.

## 11. Networking

By this point the Pi already has whatever network Imager configured in [step 1](#1-flash-the-pi). Two more networking pieces are relevant once you're actually driving the robot, both fully covered elsewhere rather than repeated here:

- **A static Ethernet link** for a reliable, WiFi-variable-free connection to your laptop — see [IP Addresses & Your Network — Direct Ethernet link](../toolkit/ip-addresses-and-your-network.md#direct-ethernet-link-a-fixed-private-network) for the Pi-side `nmcli` setup.
- **The setup hotspot fallback**, so the robot can be recovered over WiFi with no monitor attached if it ever loses its known network — see [Discovery & Dashboard](discovery-and-dashboard.md) for how it's raised and its `nmcli` permissions.

Nothing about either is specific to the *build* — both work the same regardless of how the Pi got its OS.

## 12. Clone and build the a2bot workspace

!!! pi "🤖 Pi"
    ```bash
    git clone FIXME:<your-a2bot-repo-url> ~/a2bot
    cd ~/a2bot/a2bot_ws
    rosdep install --from-paths src --ignore-src -r -y
    colcon build --symlink-install
    echo "source ~/a2bot/a2bot_ws/install/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```

    `--symlink-install` links the install directory back to the source files instead of copying them — a change to a Python file takes effect immediately on the next run, with no rebuild needed. (A change to a non-Python file that `colcon` processes at build time, like a URDF or a package's `setup.cfg`, still needs a real rebuild.)

## 13. Set up the systemd services

A2Bot's own software runs as five background services on the Pi, so no one has to SSH in and start five programs by hand every time it powers on:

| Service | Job |
|---|---|
| `a2bot-robot` | The driver stack — `serial_bridge`, `diff_drive`, `odometry`, the IMU node, the EKF, and the lidar |
| `a2bot-dashboard` | The FastAPI dashboard — live telemetry and the WiFi setup page |
| `a2bot-webui` | The Node.js web control interface set up in [step 10](#10-python-and-system-packages) |
| `a2bot-rosbridge` | The WebSocket bridge the dashboard and web control interface use to reach ROS 2 topics |
| `a2bot-gpio` | The physical recovery buttons, running independently of the ROS stack so it still works if ROS never started |

Every service here except `a2bot-webui` calls the `ros2` CLI, and a systemd unit does **not** source `~/.bashrc` the way an interactive terminal does — so each one sets `ROS_DOMAIN_ID` and `RMW_IMPLEMENTATION` explicitly in its own unit file rather than relying on it, using this robot's own domain ID:

```ini
Environment=ROS_DOMAIN_ID=X
Environment=RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Substitute this robot's own number for `X`, matching [step 7](#7-set-the-ros-domain-id) above.

See [~/.bashrc — Where this bites](../toolkit/bashrc.md#where-this-bites-in-this-project-specifically) for why `.bashrc` alone isn't enough here, and [Troubleshooting Index](../appendices/troubleshooting-index.md#a-systemd-service-comes-up-active-on-the-wrong-ros_domain_id-rmw_implementation) if a service ever comes up "active" but silently on the wrong domain.

Enable all five to start automatically at boot:

!!! pi "🤖 Pi"
    ```bash
    sudo systemctl enable --now a2bot-robot a2bot-dashboard a2bot-webui a2bot-rosbridge a2bot-gpio
    ```

    See [systemd & Services](../toolkit/systemd-and-services.md) for the general `systemctl`/`journalctl` commands to check on any of these afterward.

## 14. Flash the Arduino firmware

The udev rule from [step 9](#9-hardware-enablement) already gives the Arduino a stable `/dev/arduino` symlink the moment it's plugged in — but the board still needs to be running firmware that actually speaks the serial protocol `serial_bridge` expects before any of this talks to real hardware. See [Flashing the Arduino Firmware](../appendices/flashing-arduino-firmware.md) for the full process.

## 15. Convenience: aliases and QR stickers

A shared `~/.a2bot_aliases` file is a good place to collect shortcuts for commands you'll otherwise retype constantly — for example:

!!! pi "🤖 Pi"
    ```bash
    cat <<'EOF' >> ~/.a2bot_aliases
    alias bringup="ros2 launch a2bot_bringup robot.launch.py"
    alias driveronly="ros2 launch a2bot_bringup driver.launch.py"
    alias rebuild="cd ~/a2bot/a2bot_ws && rm -rf build install log && colcon build --symlink-install"
    EOF
    echo "source ~/.a2bot_aliases" >> ~/.bashrc
    source ~/.bashrc
    ```
    This relies on the exact same "sourced from `.bashrc` on every new terminal" mechanism as everything else on this page — see [~/.bashrc](../toolkit/bashrc.md) if that's not clear yet.

Finally, print or attach a QR code sticker to the chassis encoding this robot's connection info, so a student can scan it instead of typing an IP by hand — see [SSH & Remote Access — QR code information](../toolkit/ssh-and-remote-access.md#qr-code-information-for-the-robot) for what it should contain, and [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md#2-open-the-dashboard) for how it's actually used once the robot is running.

Next: [Setup 2 — Laptop](setup-2-laptop.md).
