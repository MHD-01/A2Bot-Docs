# Sensor Fusion / EKF

## Concept: why fuse sensors at all?

Wheel odometry (from [a2bot_driver](a2bot-driver.md)) drifts — wheel slip, tyre wear, and small errors in the assumed wheel separation all accumulate over time, and heading error is the worst offender: a small yaw error early on compounds into a large position error later. A gyroscope measures rotation rate directly and doesn't share that failure mode, but it can't measure position at all. An **Extended Kalman Filter (EKF)** fuses multiple imperfect sensors into one estimate that's better than any single source — here, via the `robot_localization` package's `ekf_node`.

## A2Bot's fusion setup

`a2bot_bringup/config/ekf.yaml` configures `ekf_filter_node` to fuse:

- **`/odom`** (wheel odometry) — position (x, y), heading (yaw), forward velocity (vx), and turn rate (vyaw).
- **`/imu/data_raw`** (the GY-85 gyro) — turn rate (vyaw) only. No orientation (the IMU driver never publishes any — see [a2bot_driver](a2bot-driver.md#imu)), and linear acceleration is excluded as too noisy on a small robot to usefully correct position.

The filter runs at 10 Hz in `two_d_mode` (the robot can't roll, pitch, or leave the ground, so those axes are locked to zero) and publishes the corrected `odom → base_link` transform itself.

!!! warning "EKF input must include yaw AND velocity, not just position"
    `robot_localization`'s `odom0_config` is a 15-element boolean array (`x, y, z / roll, pitch, yaw / vx, vy, vz / vroll, vpitch, vyaw / ax, ay, az`). Taking **only** x/y position from wheel odometry is a common misconfiguration that looks fine on paper but silently fails: the filter never fully initializes and produces zero output forever, with no error message. A2Bot's actual config correctly includes yaw (index 5), vx (index 6), and vyaw (index 11) from `/odom` alongside x and y — this is confirmed correct in the YAML. (The comment directly above it, "take X and Y position from the wheels, but NOT yaw — that's what the gyro is for," is misleading and worth a human fix: the config *does* take yaw from the wheels, and the gyro only ever contributes yaw *rate*, since it has no absolute heading to give.)

    If you ever add or modify an EKF config yourself, verify with `ros2 topic hz /odometry/filtered` — an EKF that's silently stuck outputs nothing there at all.

## The "exactly one publisher" rule, again

Just like `odometry` and the EKF can't both publish `odom → base_link` (see [a2bot_driver](a2bot-driver.md)), and just like there's no separate static transform for the lidar frame (see [Software Architecture](software-architecture.md)), the EKF's config file explicitly documents that it is the **sole publisher** of `odom → base_link` — `a2bot_driver/odometry` must run with `publish_tf: false`, which is exactly how `driver.launch.py` starts it. This "exactly one owner per transform" rule is the single most repeated design principle in this codebase; expect to see it again for `map → odom` in [SLAM & Navigation](../part3/slam-and-navigation.md).

## Running it

The EKF starts automatically as part of the driver chain — it isn't launched separately:

!!! pi "🤖 Pi"
    ```bash
    ros2 launch a2bot_bringup driver.launch.py
    ```

Check it's actually producing fused output:

!!! pi "🤖 Pi"
    ```bash
    ros2 topic hz /odometry/filtered
    ```

Next: [Closed-Loop Motion: a2bot_service](a2bot-service.md).
