# Setup 1 — Raspberry Pi

!!! important "Is the robot already built and running?"
    If yes, **skip Setup 1 entirely** and go straight to
    [Setup 2 — Laptop](setup-2-laptop.md) — you do not need to touch the
    Raspberry Pi at all. Setup 1 is only for building a robot **from
    scratch**: a bare Raspberry Pi and bare chassis with nothing
    assembled or configured yet.

This page takes a bare Raspberry Pi and bare chassis to a driving, networked robot, in the order it actually needs to happen. Each step links to the page with full concept/detail rather than repeating it here.

## 1. Flash the Pi

Flash **Ubuntu 22.04 Server** (not Desktop — headless is all the Pi needs) to an SD card using Raspberry Pi Imager. In the imager's advanced options, set the hostname, enable SSH, and set a username/password — this avoids needing a monitor and keyboard for first boot.

## 2. Wire the electronics

Assemble the chassis, wire the motors to the Arduino, and connect the Arduino, RPLidar, GY-85 IMU, and camera to the Pi's USB/I2C. See [Electronics & Hardware](../part2/electronics-hardware.md) for the full component list and pinout notes.

## 3. Flash the Arduino firmware

The Arduino must be running firmware that speaks the serial protocol described in [The Driver: a2bot_driver](../part2/a2bot-driver.md#serial_bridge) before `serial_bridge` can talk to it. See [Flashing the Arduino Firmware](../appendices/flashing-arduino-firmware.md).

## 4. Install ROS 2 Humble

!!! pi "🤖 Pi"
    Follow [Install & Source](../part1/install-and-source.md). The turtlesim package it installs isn't needed on the Pi, but the rest is identical.

## 5. Install and configure Cyclone DDS

ROS2 can run on several different DDS implementations underneath, and two machines must use the *same* one to discover each other reliably. Cyclone DDS is used here because it is what the robots themselves are already configured with — a mismatched RMW implementation between a laptop and a robot is a real, silent cause of "why can't I see any topics" that matches no error message.

!!! pi "🤖 Pi"
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

## 6. Set required Pi groups

!!! pi "🤖 Pi"
    ```bash
    sudo usermod -aG dialout,i2c,video,netdev $USER
    ```

    Log out and back in (or reboot) — group membership doesn't apply to an already-open session.

## 7. Create stable device symlinks (udev rules)

!!! pi "🤖 Pi"
    ```bash
    sudo tee /etc/udev/rules.d/99-a2bot.rules > /dev/null <<'EOF'
    SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", SYMLINK+="arduino"
    SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", SYMLINK+="rplidar"
    EOF
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    ```

    Confirm both symlinks exist after replugging the devices: `ls -l /dev/arduino /dev/rplidar`.

!!! danger "brltty may steal the Arduino before you get this far"
    If `/dev/arduino` appears then disappears seconds later, see [Troubleshooting Index](../appendices/troubleshooting-index.md#brltty-steals-the-arduino) — this is a near-universal first-build snag on Ubuntu.

## 8. Clone and build the workspace

!!! pi "🤖 Pi"
    ```bash
    git clone FIXME:<your-a2bot-repo-url> ~/a2bot
    cd ~/a2bot/a2bot_ws
    rosdep install --from-paths src --ignore-src -r -y
    colcon build --symlink-install
    echo "source ~/a2bot/a2bot_ws/install/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```

## 9. Set the ROS Domain ID

!!! pi "🤖 Pi"
    ```bash
    echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
    source ~/.bashrc
    ```

    **42, not 0** — this is a deliberate, fixed choice for this project so it never collides with another ROS 2 system's default on shared lab WiFi. It must match exactly on every machine that needs to see the robot — see [Setup 2 — Laptop](setup-2-laptop.md#4-match-the-ros-domain-id) for setting it on your laptop to match.

## 10. Set up the WiFi hotspot fallback (recommended)

`a2bot_extras`'s `wifi_manager` can bring up a provisioning hotspot (`A2Bot-Setup` at `10.42.0.1`) whenever the Pi has no known WiFi — the practical way to recover a robot that's lost network access with no monitor attached. See [Discovery & Dashboard](discovery-and-dashboard.md) for setup and the `nmcli` permissions it needs.

Next: [Setup 2 — Laptop](setup-2-laptop.md).
