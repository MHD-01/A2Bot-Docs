# Capstone: Build a Behavior

Everything up to this point has been driving A2Bot with tools someone else wrote. This capstone asks you to write your own node that gives it a behavior — combining what you learned writing nodes in [Part 1](../part1/writing-your-own-nodes.md) with what you now know about A2Bot's real topics.

## Goal

Write a single ROS 2 node that makes A2Bot do something autonomous, without using Nav2. Some options, roughly in order of difficulty:

1. **Wall follower** — subscribe to `/scan`, keep a fixed distance from the nearest wall on one side, publish `/cmd_vel`.
2. **Obstacle avoider** — drive forward on `/cmd_vel` until `/scan` reports something within a threshold distance ahead, then turn away.
3. **Patrol loop** — publish a fixed sequence of `/cmd_vel` commands (forward, turn, forward, turn...) timed to trace a square or other shape, correcting drift using `/odom`.
4. **Follow-the-gesture remix** — extend `gesture_node` (see [Discovery & Dashboard](discovery-and-dashboard.md)) with a new gesture and behavior of your own.

## What your node needs

- A subscription to whichever sensor topic your behavior depends on (`/scan` for wall-following/avoidance, `/odom` for tracking distance traveled).
- A publisher to `/cmd_vel` (`geometry_msgs/Twist`) — this is the same topic teleop and Nav2 both use, so **don't run your node at the same time as either of those**, for the same reason noted for gesture control: multiple publishers on `/cmd_vel` fight each other.
- A watchdog or explicit stop on shutdown — `diff_drive` already protects against a *stalled* command source, but your own node stopping unexpectedly (Ctrl-C, an exception) should still leave the robot stationary rather than coasting on its last command. Look at how `serial_bridge`'s `destroy_node()` sends a zero-velocity command before closing (see [a2bot_driver](../part2/a2bot-driver.md)) as the pattern to copy.

## Suggested structure

Create a new package the same way [Writing Your Own Nodes](../part1/writing-your-own-nodes.md) did:

!!! laptop "💻 Laptop"
    ```bash
    cd ~/a2bot/a2bot_ws/src
    ros2 pkg create --build-type ament_python my_capstone \
      --dependencies rclpy geometry_msgs sensor_msgs nav_msgs
    ```

Build and run it exactly like any other A2Bot node:

!!! pi "🤖 Pi"
    ```bash
    cd ~/a2bot/a2bot_ws
    colcon build --symlink-install --packages-select my_capstone
    source install/setup.bash
    ros2 run my_capstone <your_node_name>
    ```

## Test it safely

Before running on the real robot, test in Gazebo ([a2bot_description](../part2/a2bot-description.md#simulation-gazebo-classic)) — the same topics (`/cmd_vel`, `/scan`, `/odom`) exist in simulation, so a node written against them needs no changes to move from sim to hardware. Once it behaves correctly in Gazebo, run it on real A2Bot with a hand near the power switch for the first test.

## Stretch goals

- Add a `ros2 param` you can tune live (e.g. the wall-following target distance) instead of hardcoding it — see [CLI Tools & Core Concepts](../part1/cli-tools-core-concepts.md#concept-parameters).
- Record a `ros2 bag` of your best run and play it back to show it off without the robot present.
- Wire your node's status into the dashboard's `/api/status` (see [Discovery & Dashboard](discovery-and-dashboard.md)) so it shows up alongside the built-in node health checks.
