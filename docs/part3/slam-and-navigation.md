# SLAM & Navigation

## Concept: SLAM

**SLAM** (Simultaneous Localization And Mapping) builds a map of an unknown space *while* tracking the robot's position within it — the two problems are solved together because each depends on the other: you need a map to know where you are, and you need to know where you are to build an accurate map. A2Bot uses `slam_toolbox`, which matches successive lidar scans against each other and against the growing map to continuously correct drifting odometry.

SLAM adds one new transform to the tree: **`map → odom`** — the correction between where the robot *thinks* it is (`odom`, which drifts, from [Sensor Fusion / EKF](../part2/sensor-fusion-ekf.md)) and where it actually is (`map`). Exactly like every other transform in this stack, only one node may publish it at a time — see below.

## Mapping a new space

!!! pi "🤖 Pi"
    Bringup must already be running (see [Bringup & Driving](bringup-and-driving.md)) — `slam_toolbox` needs `/scan` and `odom → base_link` to exist:
    ```bash
    ros2 launch a2bot_navigation slam.launch.py
    ```

!!! laptop "💻 Laptop"
    Drive the robot around the space (teleop or gesture control) while watching the map build in RViz — add a **Map** display, topic `/map`.

Save the finished map:

!!! pi "🤖 Pi"
    ```bash
    ros2 run nav2_map_server map_saver_cli -f ~/my_new_map
    ```

## Concept: localization against a saved map

Once a space is mapped, you don't need to re-map it every time — **AMCL** (Adaptive Monte Carlo Localization) instead finds the robot's position *within* an existing, fixed map using particle filtering: many hypothesized poses are weighted against how well the lidar scan matches the map at each, converging on the real one.

!!! pi "🤖 Pi"
    Use this **instead of** `slam.launch.py`, never alongside it — both publish `map → odom`, and two publishers of one transform breaks navigation in ways that are hard to diagnose (the same rule from [Sensor Fusion / EKF](../part2/sensor-fusion-ekf.md), applied one frame up):
    ```bash
    ros2 launch a2bot_navigation localization.launch.py map:=/path/to/your_map.yaml
    ```

In RViz, use the **2D Pose Estimate** button to give AMCL a rough starting guess — without it, the robot doesn't know where in the map it started.

## Concept: autonomous navigation (Nav2)

Nav2 plans a path from the robot's current pose to a goal and drives it there, continuously replanning around obstacles the lidar sees. A2Bot runs Nav2 **alongside `slam_toolbox`** (mapping and navigating at the same time) rather than alongside AMCL — so there is deliberately no AMCL in `nav2.yaml`, and `slam_toolbox` remains the sole `map → odom` publisher.

!!! pi "🤖 Pi"
    ```bash
    # 1. bringup already running
    # 2. slam.launch.py already running (provides map -> odom and /map)
    ros2 launch a2bot_navigation nav2.launch.py
    ```

Nav2's velocity limits are set below the robot's real ceiling (max 0.15 m/s, under the ~0.165 m/s the motors can reach at their rad/s limit) and its costmap `robot_radius` uses the URDF's actual footprint (0.11–0.15 m) rather than Nav2's stock default — both deliberately tuned to this specific chassis rather than left at generic defaults.

In RViz, use the **Nav2 Goal** tool to click a destination — the robot plans and drives there on its own.

Next: [Capstone](capstone.md).
