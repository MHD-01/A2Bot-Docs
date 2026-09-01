# ROS 2 CLI Reference

A lookup table, not a tutorial — the concepts behind nodes, topics, and parameters are taught in [Part 1](../part1/cli-tools-core-concepts.md). This page is only for when you know *what* you want to check and just need the exact command, scoped to what this course actually uses repeatedly.

| Command | What it does | Real example |
|---|---|---|
| `ros2 node list` | List every currently running node | `ros2 node list` |
| `ros2 node info <node>` | Show one node's publishers, subscribers, and services | `ros2 node info /odometry` |
| `ros2 topic list` | List every currently active topic | `ros2 topic list` |
| `ros2 topic echo <topic>` | Print messages on a topic live, as they arrive | `ros2 topic echo /odom` |
| `ros2 topic hz <topic>` | Measure how often a topic is actually publishing | `ros2 topic hz /scan` |
| `ros2 launch <pkg> <file>` | Start a launch file — multiple nodes together, one command | `ros2 launch a2bot_bringup robot.launch.py` |
| `ros2 run <pkg> <executable>` | Start one specific node directly | `ros2 run a2bot_driver serial_bridge` |
| `ros2 param get <node> <param>` | Read a parameter's current value | `ros2 param get /ekf_filter_node frequency` |
| `ros2 param set <node> <param> <value>` | Change a parameter while the node is running | `ros2 param set /ekf_filter_node frequency 20.0` |
| `ros2 daemon stop` / `ros2 daemon start` | Restart the background daemon the `ros2` CLI itself relies on | `ros2 daemon stop && ros2 daemon start` |
| `colcon build --symlink-install` | Build the workspace, symlinking installed files back to source | `colcon build --symlink-install` |
| `source install/setup.bash` | Load a freshly built workspace's packages into the current shell | `source install/setup.bash` |

!!! tip "When a command "isn't working," restart the daemon first"
    `ros2 node list`/`ros2 topic list` are backed by a persistent background daemon, not a fresh check every time. If the CLI's output looks stale or wrong compared to what you know is actually running, `ros2 daemon stop && ros2 daemon start` before assuming something deeper is broken.

## Where this shows up in the course

Every page from [Part 1](../part1/cli-tools-core-concepts.md) onward uses some subset of this table — `ros2 topic echo`/`hz` especially recur throughout [Bringup & Driving](../part3/bringup-and-driving.md) and [ROS2 on Real Hardware](../part3/ros2-on-real-hardware.md) as the standard way to check whether something is actually working, not just started.
