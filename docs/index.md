# What is A2Bot?

A2Bot is a small differential-drive robot built for teaching ROS 2. It has two wheels driven by motors, a rear and front caster, a lidar, an IMU, and a camera, all controlled by a Raspberry Pi running ROS 2 Humble. It exists so students can learn robotics concepts — nodes, topics, transforms, SLAM, navigation — on hardware they can see and touch, instead of only in simulation.

## The two-machine model

Every A2Bot session involves **two computers working together**:

| Machine | Role |
|---|---|
| 🤖 **The Pi** (on the robot) | Runs the driver nodes, reads sensors, talks to the Arduino over serial, and does the actual driving. It has no screen or keyboard — you never sit at it directly. |
| 💻 **Your laptop** | Runs visualization tools (RViz), teleop keyboards, SLAM viewers, and your own code while you develop it. You SSH into the Pi, or let ROS 2's networked discovery connect the two machines' topics automatically. |

Both machines run ROS 2 nodes that discover each other over the network and exchange topics as if they were on one computer. That's the whole trick: ROS 2 doesn't care that "the lidar" and "the map viewer" are on physically different boxes, as long as they can reach each other and agree on a few network settings.

Because two machines are involved in almost everything on this site, **every command block you'll see from here on is tagged with which machine it runs on**:

!!! laptop "💻 Laptop"
    ```bash
    # A command that only makes sense on your development laptop —
    # e.g. RViz, a GUI, or your own workspace build.
    ```

!!! pi "🤖 Pi"
    ```bash
    # A command that only makes sense on the robot's Raspberry Pi —
    # e.g. reading a sensor plugged into it.
    ```

!!! both "🔗 Both"
    ```bash
    # A command you must run on BOTH machines — e.g. setting the
    # same ROS_DOMAIN_ID so they can discover each other.
    ```

This tagging is used consistently on every page. When in doubt about where to type a command, check the tag first.

## What A2Bot is built from

At a glance, five ROS 2 packages make up the robot's software (covered in depth in [Part 2](part2/software-architecture.md)):

- **a2bot_driver** — talks to the Arduino over serial, converts velocity commands to wheel speeds, and turns encoder ticks into odometry.
- **a2bot_description** — the URDF robot model: physical dimensions, sensor frames, and simulation.
- **a2bot_bringup** — the launch files that start everything together, and the EKF sensor-fusion config.
- **a2bot_navigation** — SLAM (mapping) and Nav2 (autonomous navigation) configuration.
- **a2bot_extras** — optional add-ons: a web dashboard, WiFi provisioning, and gesture control.

If you're brand new to ROS 2, don't worry about any of that yet — [Part 1](part1/install-and-source.md) starts from zero, using only a simulated turtle, before A2Bot enters the picture at all.
