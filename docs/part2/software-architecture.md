# Software Architecture

## Concept: composing a robot from packages

A ROS 2 robot is rarely one program — it's a collection of small, focused packages, each independently buildable, that a launch file starts together. This keeps any one piece (say, the motor driver) simple enough to understand and test on its own, while the launch file handles wiring them together at runtime.

## A2Bot's six packages

| Package | Job |
|---|---|
| **a2bot_driver** | Talks to the Arduino over serial; converts `/cmd_vel` into wheel commands; turns encoder feedback into odometry; reads the IMU. |
| **a2bot_description** | The URDF robot model — physical geometry, sensor frames, and Gazebo simulation config. |
| **a2bot_bringup** | Launch files that start the driver chain together, plus the EKF sensor-fusion config. |
| **a2bot_navigation** | SLAM (`slam_toolbox`) and Nav2 configuration for mapping and autonomous driving. |
| **a2bot_service** | A single node exposing move/turn/return-home/stop as request-response services, closing the loop against `/odom` instead of leaving `/cmd_vel` timing to a human. See [Closed-Loop Motion: a2bot_service](a2bot-service.md). |
| **a2bot_extras** | Optional add-ons: a web dashboard, WiFi provisioning, gesture control. Nothing in `a2bot_driver`, `a2bot_description`, `a2bot_bringup`, or `a2bot_navigation` depends on this one — though `a2bot_extras` itself does depend on `a2bot_service`, since the dashboard's movement controls call straight into its services. |

`a2bot_service` is the one package here that uses `ament_cmake` rather than `ament_python` — it needs `rosidl_generate_interfaces` to build its two custom service types, `MoveDistance` and `TurnAngle`. The rest are plain Python. There is no C++ and no `ros2_control` hardware-interface layer anywhere in this stack. If you've seen ROS 2 robots built on `ros2_control`, set that model aside; A2Bot's driver nodes talk to hardware directly.

## The node graph

```
                    ┌─────────────┐
   /cmd_vel  ─────► │ diff_drive  │ ─────► /wheel_cmd
                    └─────────────┘              │
                                                  ▼
┌──────────────┐   serial (V...)   ┌──────────────────┐
│   Arduino    │ ◄──────────────── │  serial_bridge    │
│  (firmware)  │ ────────────────► │                    │
└──────────────┘   serial (F...)   └──────────────────┘
                                                  │
                                       /joint_states
                                                  │
                                                  ▼
                                        ┌──────────────┐
                                        │  odometry    │ ──► /odom, odom→base_link (or feeds the EKF)
                                        └──────────────┘

              /imu/data_raw ◄───── imu (GY-85)

   /odom + /imu/data_raw ──────► ekf_filter_node ──► odom→base_link (fused)
```

`diff_drive`, `serial_bridge`, `odometry`, and `imu` are all separate nodes in `a2bot_driver`, each doing one job — exactly the small-focused-node philosophy from [Part 1](../part1/writing-your-own-nodes.md), just applied to real hardware instead of a simulated turtle. Full detail on each is in [The Driver: a2bot_driver](a2bot-driver.md).

## The serial protocol, in outline

`serial_bridge` is the **only** node allowed to open the Arduino's serial port — everything else reaches it only through ROS topics. The wire format is a small, fixed text protocol:

| Direction | Format | Meaning |
|---|---|---|
| Pi → Arduino | `V<left_rad_s>,<right_rad_s>\n` | Target angular velocity for each wheel, rad/s |
| Arduino → Pi | `F<l_pos>,<r_pos>,<l_vel>,<r_vel>\n` | Encoder position (rad) and velocity (rad/s) per wheel, sent continuously at 50 Hz |

Serial port: `/dev/arduino` (a udev symlink, not a raw device name — see [Electronics & Hardware](electronics-hardware.md)), at **115200 baud**. Full detail, including the DTR-reset dance and watchdog behavior, is in [The Driver: a2bot_driver](a2bot-driver.md#serial_bridge).

## How it starts

`a2bot_bringup`'s launch files compose the others rather than duplicating them — each stays independently runnable:

- `driver.launch.py` — the three `a2bot_driver` motion nodes plus the IMU and EKF.
- `a2bot_description`'s `description.launch.py` — `robot_state_publisher`, turning the URDF into `tf`.
- `robot.launch.py` (in `a2bot_bringup`) — includes both of the above, plus the lidar node and `a2bot_service`'s `service_server` node.

Deliberately **absent**: any `static_transform_publisher` for the lidar or IMU frames. The URDF already defines `base_link → lidar_link` and `base_link → imu_link`, published by `robot_state_publisher`; a second, separate transform publisher for the same link would create two publishers of one transform — a source of hard-to-diagnose tf conflicts that recurs throughout this stack's design (see [Sensor Fusion / EKF](sensor-fusion-ekf.md) for the same principle applied to `odom → base_link`).

Next: [The Driver: a2bot_driver](a2bot-driver.md).
