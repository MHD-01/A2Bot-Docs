# Bringup & Driving

## Concept: bringup

"Bringup" is the standard ROS 2 term for the single launch command that starts everything needed to consider a robot "up" — as opposed to launching each node by hand in separate terminals, which is how [Part 1](../part1/writing-your-own-nodes.md) started nodes individually to make each concept visible.

## Start the robot

!!! pi "🤖 Pi"
    ```bash
    ros2 launch a2bot_bringup robot.launch.py
    ```

This starts the full driver chain (`serial_bridge`, `diff_drive`, `odometry`, `imu`, the EKF — see [a2bot_driver](../part2/a2bot-driver.md)), `robot_state_publisher` (see [a2bot_description](../part2/a2bot-description.md)), the lidar node, and `a2bot_service`'s `service_server` (see below). After this, the robot can be driven and its full `tf` tree exists, but no map or navigation is running yet — that's [SLAM & Navigation](slam-and-navigation.md).

## Visualize it

!!! laptop "💻 Laptop"
    ```bash
    rviz2
    ```

    Set the Fixed Frame to `odom`, then add a **RobotModel** display and a **LaserScan** display (topic `/scan`). If the robot model doesn't appear, re-check that you built `a2bot_description` locally on the laptop — see the note in [Setup 2](setup-2-laptop.md#3-build-the-workspace-locally).

## Drive it manually

The standard ROS 2 keyboard teleop node works unchanged — it just publishes `/cmd_vel`, exactly like it drove the turtle in [Part 1](../part1/cli-tools-core-concepts.md):

!!! laptop "💻 Laptop"
    ```bash
    ros2 run teleop_twist_keyboard teleop_twist_keyboard
    ```

For smoother, less abrupt manual driving, `a2bot_navigation` provides a standalone velocity smoother that applies acceleration limits between teleop and the motors:

!!! pi "🤖 Pi"
    ```bash
    ros2 launch a2bot_navigation smoother.launch.py
    ```

!!! laptop "💻 Laptop"
    ```bash
    ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r cmd_vel:=cmd_vel_nav
    ```

    Teleop now publishes to `/cmd_vel_nav`, the smoother publishes the smoothed result to `/cmd_vel`, and `diff_drive` never sees a sudden jump in requested speed.

## Drive it with closed-loop commands

Teleop leaves it to you to watch the robot and release the key at the right moment. For "move exactly 1 m" or "turn exactly 90°" instead, `a2bot_service` (started automatically above) exposes that as a request that only returns once the odometry confirms the target was reached:

!!! laptop "💻 Laptop"
    ```bash
    ros2 run a2bot_service move_client 1.0 --speed 0.2
    ros2 run a2bot_service turn_client 90 --speed 0.5
    ros2 run a2bot_service return_home_client
    ```

See [Closed-Loop Motion: a2bot_service](../part2/a2bot-service.md) for the full service list, and a known limitation of `return_home` after a stopped or timed-out move.

## Simulating instead

No physical robot handy? The same launch pattern works in Gazebo — see [a2bot_description](../part2/a2bot-description.md#simulation-gazebo-classic):

!!! laptop "💻 Laptop"
    ```bash
    ros2 launch a2bot_description sim.launch.py
    ```

    Then drive it exactly the same way: `ros2 run teleop_twist_keyboard teleop_twist_keyboard`.

Next: [Discovery & Dashboard](discovery-and-dashboard.md).
