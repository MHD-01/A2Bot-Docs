# The Robot Model: a2bot_description

## Concept: URDF and why a robot needs one

A **URDF** (Unified Robot Description Format) is an XML file describing a robot's physical structure: its rigid parts (**links**), how they connect and move relative to each other (**joints**), and where sensors sit. ROS 2 tools rely on it for two different jobs:

- `robot_state_publisher` reads the URDF and publishes the corresponding `tf` transform tree, so any node can ask "where is the lidar relative to the base?" without hardcoding the answer.
- RViz and Gazebo read the same file to draw the robot and, in Gazebo's case, physically simulate it.

A2Bot's URDF lives at `a2bot_description/urdf/a2bot.urdf` and is **plain URDF, not xacro** — no macros or generated values. It is the single source of truth for every physical dimension referenced anywhere in this documentation; where any launch file, node default, or piece of prose disagrees with it, the URDF wins (see the [driver page](a2bot-driver.md) for the one place this actually happened).

## Links, joints, and frames

| Link | Parent joint | Notes |
|---|---|---|
| `base_link` | — (root) | Body origin. Visual mesh is the TurtleBot3 Burger base; collision is a simplified box. |
| `left_wheel` | `left_wheel_joint` (continuous) | At `y = +0.08 m`, radius 0.033 m |
| `right_wheel` | `right_wheel_joint` (continuous) | At `y = -0.08 m`, radius 0.033 m |
| `caster_wheel` | `caster_joint` (fixed) | Passive rear caster, collision-only geometry |
| `imu_link` | `imu_joint` (fixed) | GY-85 mounting point |
| `lidar_link` | `lidar_joint` (fixed) | RPLidar mounting point |

Fixed joints (caster, IMU, lidar) are published once by `robot_state_publisher` and never change. The two wheel joints are `continuous` (unbounded rotation) and are driven from live `/joint_states` messages — this is how the wheel meshes actually spin in RViz as the robot drives.

## No `base_footprint` — on purpose

Many ROS 2 robot examples (including stock TurtleBot3) add a `base_footprint` link — a ground-projected point directly below `base_link`, used as the nav stack's nominal robot origin. **A2Bot deliberately has no `base_footprint`.** In `tf`, a frame may only have one parent; `odometry` (or the EKF) already publishes `odom → base_link` directly, and adding `base_footprint → base_link` on top would give `base_link` two parents, which breaks the tf tree. This is not a bug or an oversight — every navigation config in `a2bot_navigation` is written to use `base_link` as the robot base frame directly (see [SLAM & Navigation](../part3/slam-and-navigation.md)).

## The lidar frame is `lidar_link`, not `laser`

The stock `rplidar_ros` launch file defaults its `frame_id` to `"laser"`. A2Bot's `robot.launch.py` explicitly overrides this to `lidar_link`, matching the URDF's actual link name — get this wrong and `/scan` messages arrive stamped with a frame that doesn't exist in the `tf` tree, which SLAM and Nav2 both depend on silently working.

## Publishing the model

`robot_state_publisher` reads the URDF file as plain text (not a file path) and republishes it as the `robot_description` parameter, then derives `tf` from it:

!!! pi "🤖 Pi"
    ```bash
    ros2 launch a2bot_description description.launch.py
    ```

In practice this is almost always started as part of `a2bot_bringup/robot.launch.py` rather than alone — see [Bringup & Driving](../part3/bringup-and-driving.md).

## Simulation (Gazebo Classic)

The same URDF file also carries a block of `<gazebo>` tags, ignored entirely by `robot_state_publisher` and RViz, that let it run in simulation without any hardware:

!!! laptop "💻 Laptop"
    ```bash
    ros2 launch a2bot_description sim.launch.py
    ```

A single Gazebo plugin, `libgazebo_ros_diff_drive`, stands in for **three** real-robot nodes at once (`diff_drive` + `serial_bridge` + `odometry`) — in simulation there's no serial port to bridge and no encoder noise to integrate, so one plugin covers all three, reading `/cmd_vel` and publishing `/odom` directly from physics. Simulated ray-cast and IMU plugins similarly stand in for the real `rplidar_node` and `imu` node. Note there is **no EKF running in simulation** — the EKF's entire purpose is fusing away *real* sensor noise, and Gazebo's simulated odometry has none to correct.

Next: [Sensor Fusion / EKF](sensor-fusion-ekf.md).
