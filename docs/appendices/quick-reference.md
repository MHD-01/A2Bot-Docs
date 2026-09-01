# Quick Reference

## Locked physical parameters

Confirmed against `a2bot_description/urdf/a2bot.urdf` (the source of truth) and, where relevant, cross-checked against the value actually passed at launch time.

| Parameter | Value |
|---|---|
| Wheel radius | `0.033 m` |
| Wheel separation | `0.160 m` (wheels at y = ±0.08 m from `base_link`) |
| Wheel joint names | `left_wheel_joint`, `right_wheel_joint` |
| Max wheel speed | `5.0 rad/s` (`max_wheel_rad_s` in `diff_drive`) |
| `base_footprint` | **Does not exist** — deliberate, see [a2bot_description](../part2/a2bot-description.md) |
| Lidar frame | `lidar_link` (not `laser`) |
| IMU frame | `imu_link` |

!!! warning "Known stale value — do not use 0.225"
    `odometry.py`'s parameter default and a comment in `diff_drive.py`/`driver.launch.py` reference **0.225 m**, left over from an earlier chassis revision. It is not what the robot actually runs with (`driver.launch.py` overrides it to 0.16 at launch) and should never be reproduced. **0.160 m is correct.**

## Serial protocol (Pi ↔ Arduino)

| | |
|---|---|
| Device symlink | `/dev/arduino` |
| Baud rate | `115200` |
| Pi → Arduino | `V<left_rad_s>,<right_rad_s>\n` |
| Arduino → Pi | `F<l_pos>,<r_pos>,<l_vel>,<r_vel>\n` at 50 Hz (positions in radians, velocities in rad/s) |

## Networking

| | |
|---|---|
| `ROS_DOMAIN_ID` | `X`, this robot's own number (must match on every machine — **not** the ROS 2 default of 0, and **not** shared with any other robot) |
| RMW implementation | Not pinned to a specific value anywhere in this repo — use whatever your ROS 2 Humble install defaults to, or check with `echo $RMW_IMPLEMENTATION` |
| WiFi hotspot SSID | `a2botX-setup` |
| WiFi hotspot password | `a2bot123` |
| WiFi hotspot address | `10.42.0.X` |
| Dashboard | `http://<robot-ip>:8888/dashboard` |
| Direct Ethernet — Pi | `10.0.0.10X/24` |
| Direct Ethernet — laptop | `10.0.0.250/24` |

`X` above is this robot's own number — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full naming convention.

## USB device pinouts (udev)

| Device | Chip | USB ID | Symlink |
|---|---|---|---|
| Arduino Nano (clone) | CH340 | `1a86:7523` | `/dev/arduino` |
| RPLidar A1 | CP2102 | `10c4:ea60` | `/dev/rplidar` |

## GY-85 IMU I2C addresses

| Chip | Address | Byte order | Used? |
|---|---|---|---|
| ADXL345 (accelerometer) | `0x53` | Little-endian | Yes |
| ITG-3200 (gyroscope) | `0x68` or `0x69` | Big-endian | Yes |
| Magnetometer | `0x0d` or `0x1e` | — | No (deliberately unused) |

## Required Pi user groups

| Group | For |
|---|---|
| `dialout` | Serial (Arduino, RPLidar) |
| `i2c` | IMU |
| `video` | Camera |
| `netdev` | WiFi management (`nmcli`) |

Each requires logging out and back in (or a reboot) to take effect.

## Most-used commands

| Task | Command | Machine |
|---|---|---|
| Full bringup | `ros2 launch a2bot_bringup robot.launch.py` | 🤖 Pi |
| Driver chain only | `ros2 launch a2bot_bringup driver.launch.py` | 🤖 Pi |
| Simulate | `ros2 launch a2bot_description sim.launch.py` | 💻 Laptop |
| Map a new space | `ros2 launch a2bot_navigation slam.launch.py` | 🤖 Pi |
| Localize on a saved map | `ros2 launch a2bot_navigation localization.launch.py map:=<file>.yaml` | 🤖 Pi |
| Navigate autonomously | `ros2 launch a2bot_navigation nav2.launch.py` | 🤖 Pi |
| Drive manually | `ros2 run teleop_twist_keyboard teleop_twist_keyboard` | 💻 Laptop |
| Web dashboard | `ros2 run a2bot_extras dashboard` | 🤖 Pi |
| Stale build reset | `rm -rf build install log && colcon build --symlink-install` | 🔗 Both |
