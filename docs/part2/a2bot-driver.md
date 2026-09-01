# The Driver: a2bot_driver

`a2bot_driver` is four nodes that together turn a `/cmd_vel` command into wheel motion and turn encoder ticks back into a pose estimate. Each does one job, communicating only over ROS topics — none of them import or call each other directly.

## Concept: differential drive kinematics

A differential-drive robot steers by spinning its two wheels at different speeds — no separate steering mechanism. Given a desired forward speed `v` and turn rate `ω`, the two wheels' surface speeds are:

```
v_left  = v - ω · (wheel_separation / 2)
v_right = v + ω · (wheel_separation / 2)
```

...and each wheel's *angular* speed (what a motor controller actually wants, in rad/s) is that surface speed divided by the wheel radius. This is standard differential-drive math — nothing here is A2Bot-specific yet.

## `diff_drive`: `/cmd_vel` → `/wheel_cmd`

Subscribes to `/cmd_vel` (`geometry_msgs/Twist`), applies the kinematics above, and publishes `/wheel_cmd` (`sensor_msgs/JointState`, using `.velocity` for each of `left_wheel_joint` / `right_wheel_joint`). It also runs a **watchdog**: if no `/cmd_vel` message arrives for `cmd_timeout_s` (default 0.5 s), it publishes zero velocity — so a dropped network connection or a closed teleop window stops the robot instead of leaving it running at its last command.

!!! warning "Stale wheel_separation value in the code — flagged, not carried forward"
    The **URDF is the source of truth**, and it places the wheels at `y = ±0.08 m`, i.e. a separation of **0.160 m** — this is also the value actually passed at launch time in `a2bot_bringup/launch/driver.launch.py`, so it's what the robot runs with today.

    However, `diff_drive.py` and `driver.launch.py` both carry a **stale comment** claiming "separation is 0.225 because the URDF places the wheels at y = ±0.1125" — that was true of an earlier chassis revision and was never updated after the URDF changed. Worse, `odometry.py`'s `declare_parameter('wheel_separation', 0.225)` uses the stale value as its **default**, though `driver.launch.py` always overrides it with 0.16 at launch, so the robot's actual behavior is unaffected today. This is worth a human cleanup pass (fix the comments, fix the unused default) since a future launch file that forgets to pass the override would silently regress to the wrong value. **0.160 m is correct and is used everywhere in this documentation.**

## `serial_bridge`: the only node that touches the Arduino {#serial_bridge}

`serial_bridge` owns `/dev/arduino` at **115200 baud** exclusively — every other node reaches the hardware only via `/wheel_cmd` and `/joint_states`. Two responsibilities:

- **Writing**: on every `/wheel_cmd` message, sends `V<left_rad_s>,<right_rad_s>\n` down the wire.
- **Reading**: the Arduino sends `F<l_pos>,<r_pos>,<l_vel>,<r_vel>\n` continuously at 50 Hz (positions in radians, velocities in rad/s). A 50 Hz timer polls the port, buffers incoming bytes, and only parses complete newline-terminated lines — so a read landing mid-transmission never gets parsed as garbage. Complete lines are republished as `/joint_states`.

Two hardware quirks worth knowing:

- **Opening the serial port resets the Arduino** (via DTR), which then spends ~2 seconds in its bootloader printing garbage. `serial_bridge` deliberately toggles DTR, waits it out, and clears the input buffer before treating the connection as ready.
- **On shutdown**, it explicitly sends `V0.0000,0.0000\n` before closing the port. Without this, the Arduino keeps driving at its last commanded speed until its own onboard watchdog trips — so a bare Ctrl-C while the robot is moving would otherwise leave it rolling.

## `odometry`: dead reckoning

Subscribes to `/joint_states`, integrates the wheel angle deltas into an (x, y, θ) pose using the same differential-drive math as `diff_drive`, and publishes `/odom` plus (optionally) the `odom → base_link` transform. Wheel odometry **drifts** — slip and imprecision accumulate as error over time — which is exactly what SLAM later corrects using the lidar (see [SLAM & Navigation](../part3/slam-and-navigation.md)).

This node's `publish_tf` parameter is set to **`false`** in `driver.launch.py` — the EKF publishes `odom → base_link` instead. Two publishers of the same transform is a recurring failure mode across this whole stack (see [Software Architecture](software-architecture.md) and [Sensor Fusion / EKF](sensor-fusion-ekf.md)), so exactly one of `odometry` or the EKF must own it at a time.

## `imu`: the GY-85 driver {#imu}

Reads the ADXL345 accelerometer and ITG-3200 gyroscope described in [Electronics & Hardware](electronics-hardware.md), verifying both chip IDs at startup before trusting their data. Publishes `/imu/data_raw` (`sensor_msgs/Imu`) — **raw data only**: `orientation_covariance[0]` is set to `-1.0`, the ROS convention meaning "orientation not provided, ignore this field." The magnetometer chip on the same board is deliberately never read: for a ground robot the gyro already gives yaw rate directly, and magnetometers are badly disturbed indoors by motors and steel furniture — a bad heading reference is worse than none.

!!! warning "The robot must be stationary for ~1 second after this node starts"
    The gyro reads a small non-zero rotation at rest; this node measures and subtracts that bias by averaging the first `calibration_samples` (default 200) readings. Moving the robot during this window bakes a wrong bias in, which later shows up as slow, hard-to-trace heading drift.

## Running the driver chain standalone

!!! pi "🤖 Pi"
    ```bash
    ros2 launch a2bot_bringup driver.launch.py
    ```

This starts `serial_bridge`, `diff_drive`, `odometry`, the `imu` node, and the EKF — everything needed to drive and see odometry, with no lidar or SLAM yet. See [Bringup & Driving](../part3/bringup-and-driving.md) for driving it.

Next: [The Robot Model: a2bot_description](a2bot-description.md).
