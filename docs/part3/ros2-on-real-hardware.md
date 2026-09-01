# ROS2 on Real Hardware

Part 1 covered ROS 2 concepts using a simulated turtle running entirely on one machine. This page covers what's genuinely different once real hardware and two networked machines are involved, before you actually drive the robot in the next page.

## Concept: discovery over a real network

In Part 1, every node was on one machine, so ROS 2's discovery (nodes automatically finding each other) had nothing to cross. With A2Bot, the Pi and your laptop are separate machines exchanging discovery traffic over actual WiFi or Ethernet — which means real-world network behavior now matters in a way it never did with turtlesim:

- Both machines must share the same **`ROS_DOMAIN_ID`** (`42` for A2Bot — see [Setup 1](setup-1-raspberry-pi.md#9-set-the-ros-domain-id) or [Setup 2](setup-2-laptop.md#4-match-the-ros-domain-id)), so ROS 2 doesn't lump you in with every other domain-0 ROS system on the same network.
- Discovery relies on **UDP multicast**, which some networks quietly block or isolate even while normal ping/SSH traffic works fine — see the AP-isolation note in [Setup 2](setup-2-laptop.md#5-get-on-the-same-network-as-the-robot).
- Unlike turtlesim, hardware nodes fail in ways a simulator never does: a dropped USB cable, a stalled I2C bus, a serial port claimed by another process. Expect to spend more time reading log output here than you did in Part 1.

## Concept: sensors have real, imperfect data

Turtlesim's pose is exact. Real sensors are not:

- **Wheel odometry drifts** — see [a2bot_driver](../part2/a2bot-driver.md).
- **The IMU needs a stationary calibration window** at startup — see [a2bot_driver](../part2/a2bot-driver.md#imu).
- **The lidar has a dead zone** (objects closer than ~0.15–0.2 m don't register reliably) — relevant once you reach [SLAM & Navigation](slam-and-navigation.md).

None of this is a bug to fix; it's the normal condition of working with hardware, and it's exactly why [Sensor Fusion / EKF](../part2/sensor-fusion-ekf.md) and SLAM's scan-matching exist — to correct for it rather than assume it away.

## Checking the driver chain is healthy

Before driving, confirm the core topics are actually flowing — silence on any of these is the first sign something's wrong:

!!! pi "🤖 Pi"
    ```bash
    ros2 topic hz /joint_states     # from serial_bridge — should be ~50 Hz
    ros2 topic hz /odom             # from odometry
    ros2 topic hz /imu/data_raw     # from the imu node
    ```

!!! laptop "💻 Laptop"
    ```bash
    ros2 topic echo /odometry/filtered   # from the EKF, once running
    ```

If `/joint_states` is silent, check the serial connection (`ls -l /dev/arduino`, then re-check [Troubleshooting Index](../appendices/troubleshooting-index.md)) before looking anywhere else — everything else in the driver chain depends on it.

Next: [Bringup & Driving](bringup-and-driving.md).
