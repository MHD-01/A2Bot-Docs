# Bringup & Driving

## Concept: bringup

"Bringup" is the standard ROS 2 term for the single launch command that starts everything needed to consider a robot "up" — as opposed to launching each node by hand in separate terminals, which is how [Part 1](../part1/writing-your-own-nodes.md) started nodes individually to make each concept visible.

## Before you start: stop the background services

On a set-up Pi, six [systemd](../toolkit/systemd-and-services.md) services — `a2bot-robot`, `a2bot-dashboard`, `a2bot-rosbridge`, `a2bot-webui`, `a2bot-gpio`, and the `ros2-ready` gate they all depend on (unit files under `~/a2bot/services/services_files/`) — start automatically at every boot, specifically so the robot works the instant it powers on with no one needing to SSH in first. `a2bot-robot` in particular is already running the exact launch file this page is about to have you start by hand: it has already opened the Arduino's serial port and started the lidar motor.

Launching `robot.launch.py` manually on top of that doesn't add a second robot — it fails to open the already-claimed serial port, or fights the running service for `/cmd_vel` and the lidar. Stop and disable the services first:

!!! pi "🤖 Pi"
    ```bash
    sudo systemctl stop a2bot-robot a2bot-dashboard a2bot-webui a2bot-rosbridge ros2-ready a2bot-gpio
    sudo systemctl disable a2bot-robot a2bot-dashboard a2bot-webui a2bot-rosbridge ros2-ready a2bot-gpio
    ```

    `stop` ends them for this boot only — the next reboot brings them straight back. `disable` is the separate step that stops them from auto-starting at the *next* boot; it does **not** stop something already running, which is why this page runs both together. Confirm with `systemctl status a2bot-robot` — it should read `inactive (dead)`.

See [systemd & Services](../toolkit/systemd-and-services.md) for the full command reference (including `journalctl` for reading a service's logs) and what each unit file actually does. Once you're done manually launching things for this page, `sudo systemctl enable --now <service>` brings each one back to its normal always-on behavior.

## Start the robot

!!! pi "🤖 Pi"
    ```bash
    ros2 launch a2bot_bringup robot.launch.py
    ```

This starts the full driver chain (`serial_bridge`, `diff_drive`, `odometry`, `imu`, the EKF — see [a2bot_driver](../part2/a2bot-driver.md)), `robot_state_publisher` (see [a2bot_description](../part2/a2bot-description.md)), the lidar node, the Pi camera (see below), and `a2bot_service`'s `service_server` (see below). After this, the robot can be driven and its full `tf` tree exists, but no map or navigation is running yet — that's [SLAM & Navigation](slam-and-navigation.md).

## Visualize it

!!! laptop "💻 Laptop"
    ```bash
    rviz2
    ```

    Set the Fixed Frame to `odom`, then add a **RobotModel** display and a **LaserScan** display (topic `/scan`). If the robot model doesn't appear, re-check that you built `a2bot_description` locally on the laptop — see the note in [Setup 2](setup-2-laptop.md#3-build-the-workspace-locally).

To also see the camera feed, add an **Image** display and point it at `/camera/image_raw/compressed` (check "compressed" under that display's transport hint) — or `/camera/image_raw` for the uncompressed stream. The same feed is also viewable without RViz at all, from the [dashboard](discovery-and-dashboard.md#the-dashboard).

!!! warning "camera_link isn't in the URDF yet"
    `robot.launch.py` stamps every camera frame with `frame_id: camera_link`, but the URDF has no `camera_link` — so that frame doesn't exist anywhere in the `tf` tree. A plain **Image** display doesn't care, since it just shows pixels with no `tf` lookup involved. Anything that *does* need `tf` for the camera (a **Camera** display trying to project the image in 3D, or fusing it with the point cloud) will drop it silently until `camera_link` is added to the URDF the same way `lidar_link` and `imu_link` already are — see [a2bot_description](../part2/a2bot-description.md).

## Drive it manually

The standard ROS 2 keyboard teleop node works unchanged — it just publishes `/cmd_vel`, exactly like it drove the turtle in [Part 1](../part1/cli-tools-core-concepts.md):

!!! laptop "💻 Laptop"
    ```bash
    ros2 run teleop_twist_keyboard teleop_twist_keyboard
    ```

## Drive it with closed-loop commands

Teleop leaves it to you to watch the robot and release the key at the right moment. For "move exactly 1 m" or "turn exactly 90°" instead, `a2bot_service` (started automatically above) exposes that as a request that only returns once the odometry confirms the target was reached:

!!! laptop "💻 Laptop"
    ```bash
    ros2 run a2bot_service move_client 1.0 --speed 0.2
    ros2 run a2bot_service turn_client 90 --speed 0.5
    ros2 run a2bot_service return_home_client
    ```

See [Closed-Loop Motion: a2bot_service](../part2/a2bot-service.md) for the full service list, and a known limitation of `return_home` after a stopped or timed-out move.

Next: [Discovery & Dashboard](discovery-and-dashboard.md).
