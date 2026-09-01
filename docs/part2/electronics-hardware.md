# Electronics & Hardware

## Concept: why a Pi *and* an Arduino?

Many hobby robots use only a microcontroller, or only a single-board computer. A2Bot uses both, each doing what it's better at:

- The **Arduino Nano** runs a tight, real-time loop reading wheel encoders and driving motor PWM — the kind of low-latency, no-operating-system work microcontrollers excel at.
- The **Raspberry Pi** runs Ubuntu and ROS 2 — ROS 2 nodes need a real OS, filesystem, and network stack, none of which a bare Arduino has.

The two talk over a USB-serial link with a small text protocol, detailed in [The Driver: a2bot_driver](a2bot-driver.md).

## The chassis and sensors

A2Bot's geometry is based on the TurtleBot3 Burger reference design (the URDF meshes and body dimensions come from it), with these confirmed components:

| Component | Detail |
|---|---|
| Drive | 2 DC gear motors, differential drive, rear passive caster |
| Wheel radius | **0.033 m** |
| Wheel separation | **0.160 m** (wheels at y = ±0.08 m from `base_link`) |
| Motor controller | Arduino Nano (clone), USB-serial to the Pi |
| Lidar | RPLidar A1, USB-serial to the Pi (`/dev/rplidar`) |
| IMU | GY-85 breakout — **three separate I2C chips**, see below |
| Camera | USB or CSI camera via `camera_ros` |
| Compute | Raspberry Pi 4, Ubuntu 22.04, ROS 2 Humble |

## The GY-85 is three chips, not one

If you've used a single-chip IMU before (an MPU-6050, MPU-9265, etc.), the GY-85 is a different animal: it's a breakout board carrying **three independent sensor chips**, each with its own I2C address and register map:

| Chip | Address | Function | Byte order |
|---|---|---|---|
| ADXL345 | `0x53` | Accelerometer | Little-endian |
| ITG-3200 | `0x68` (or `0x69`) | Gyroscope | **Big-endian** |
| QMC5883L-type | `0x0d` (or `0x1e`) | Magnetometer | Not used |

The two chips actually in use have **opposite byte order on the same physical board** — a driver written assuming one convention will read the other chip's data backwards. This matters because a byte-order mistake doesn't crash or error: it produces plausible-looking but wrong numbers. A2Bot's `imu.py` driver ([details here](a2bot-driver.md#imu)) verifies both chip IDs at startup (`ADXL345 DEVID` should read `0xe5`, `ITG-3200 WHO_AM_I` should read `0x68` or `0x69`) specifically to catch a mis-wired or dead chip before trusting its data. The magnetometer is deliberately unused — see that page for why.

## Wiring & power

- **Power**: 5V/3A minimum for the Pi. Undervoltage causes flaky WiFi and USB behavior that looks like a software bug. Check with:

!!! pi "🤖 Pi"
    ```bash
    dmesg | grep -i voltage
    ```

    Any "under-voltage detected" line means the symptom you're chasing may just be the power supply.

- **Required Pi user groups** — each lets a normal user account access a piece of hardware without `sudo`, and **each requires logging out and back in (or a reboot) to take effect**, not just running the command:

| Group | Grants access to |
|---|---|
| `dialout` | Serial ports (Arduino, RPLidar) |
| `i2c` | The IMU |
| `video` | The camera |
| `netdev` | WiFi management via `nmcli` |

!!! pi "🤖 Pi"
    ```bash
    sudo usermod -aG dialout,i2c,video,netdev $USER
    # then log out and back in, or reboot
    ```

    New to combined short flags like `-aG`? See [Ubuntu Terminal Basics](../toolkit/ubuntu-terminal-basics.md).

## Stable device names (udev)

Linux assigns `/dev/ttyUSB0`-style names by **plug-in order**, not by device identity — plug the Arduino in after the lidar one day and the names swap. A2Bot's code deliberately never references a raw `/dev/ttyUSB*` name; it always uses fixed symlinks (`/dev/arduino`, `/dev/rplidar`) created by udev rules matching each device's USB vendor/product ID:

| Device | USB ID | Symlink |
|---|---|---|
| Arduino (CH340 chip) | `1a86:7523` | `/dev/arduino` |
| RPLidar (CP2102 chip) | `10c4:ea60` | `/dev/rplidar` |

These rule files live on the Pi's filesystem under `/etc/udev/rules.d/`, outside the git-tracked workspace, so they aren't shipped in this repository — see [Setup 1 — Raspberry Pi](../part3/setup-1-raspberry-pi.md) for creating them from scratch.

!!! warning "brltty steals the Arduino"
    Ubuntu ships a service called `brltty` (braille display support) that misidentifies CH340 USB-serial adapters as braille hardware and silently claims the device seconds after it's plugged in — `/dev/arduino` briefly appears, then vanishes, with no obvious error. See [Troubleshooting Index](../appendices/troubleshooting-index.md) for the fix.

Next: [Software Architecture](software-architecture.md).
