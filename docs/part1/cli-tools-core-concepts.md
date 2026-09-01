# CLI Tools & Core Concepts

This page covers the core ideas every ROS 2 program is built from, using `turtlesim` so you can see each concept without needing A2Bot. Everything here is standard ROS 2 — none of it is A2Bot-specific.

## Concept: nodes and topics

A **node** is a single running program that does one job. Nodes talk to each other by publishing and subscribing to **topics** — named channels carrying a specific message type. A publisher doesn't know who's listening, and a subscriber doesn't know who's publishing; they're only coupled by the topic name and message type. This decoupling is why ROS 2 systems scale to dozens of independent programs.

Start the simulator and a keyboard teleop node — two separate nodes, talking over a topic:

!!! laptop "💻 Laptop"
    ```bash
    ros2 run turtlesim turtlesim_node
    ```

!!! laptop "💻 Laptop"
    ```bash
    ros2 run turtlesim turtle_teleop_key
    ```

With the second terminal focused, use the arrow keys to drive the turtle. Every keypress is being published as a message; the simulator node is subscribed and moving the turtle in response.

## Inspect what's running

!!! laptop "💻 Laptop"
    ```bash
    ros2 node list
    ros2 topic list
    ros2 topic echo /turtle1/cmd_vel
    ros2 topic hz /turtle1/pose
    ```

`ros2 topic echo` prints every message flowing over a topic live — the single most useful debugging command in ROS 2. `ros2 topic hz` measures how often a topic is actually publishing, which is how you catch a node that's silently stalled.

## Concept: services

A topic is one-way and continuous. A **service** is the opposite: a one-shot request/response call, like a function call across process boundaries. Use a service when you want an answer, not a stream.

!!! laptop "💻 Laptop"
    ```bash
    ros2 service list
    ros2 service call /spawn turtlesim/srv/Spawn "{x: 2, y: 2, theta: 0, name: 'turtle2'}"
    ```

This spawns a second turtle and returns immediately with the result — no ongoing subscription needed.

## Concept: parameters

**Parameters** are per-node configuration values — numbers, strings, or booleans a node reads at startup or lets you change live, without recompiling.

!!! laptop "💻 Laptop"
    ```bash
    ros2 param list
    ros2 param get /turtlesim background_r
    ros2 param set /turtlesim background_r 150
    ```

## Concept: actions

An **action** is for long-running tasks that report progress and can be cancelled — a middle ground between a topic (continuous, no completion) and a service (instant, no progress). Turtlesim's `RotateAbsolute` is a simple example:

!!! laptop "💻 Laptop"
    ```bash
    ros2 action list
    ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: 1.57}"
    ```

Watch the terminal print feedback as the turtle rotates, rather than jumping straight to a final result.

## Concept: logging

Nodes report their own status with log messages at different severities (`debug`, `info`, `warn`, `error`, `fatal`). Watch them live:

!!! laptop "💻 Laptop"
    ```bash
    ros2 topic echo /rosout
    ```

In practice you'll usually just read a node's own terminal output (each node prints its logs to the terminal it was started in), but `/rosout` aggregates every node's logs onto one topic — useful when many nodes are running across two machines and you can't watch every terminal.

## Concept: launch files

Starting each node in its own terminal doesn't scale past two or three nodes. A **launch file** is a Python script that starts a whole group of nodes together, with their parameters and remappings pre-configured.

!!! laptop "💻 Laptop"
    ```bash
    ros2 launch turtlesim multisim.launch.py
    ```

This one command starts two independent turtlesim instances — the same job would take two manually-coordinated terminals otherwise. A2Bot's entire software stack starts this way; see [Software Architecture](../part2/software-architecture.md).

## Concept: ros2 bag

A **bag** records every message on chosen topics to disk, so you can replay a session later exactly as it happened — invaluable for debugging something that only happened once.

!!! laptop "💻 Laptop"
    ```bash
    ros2 bag record /turtle1/cmd_vel /turtle1/pose
    ```

Press Ctrl+C to stop, then replay it:

!!! laptop "💻 Laptop"
    ```bash
    ros2 bag play rosbag2_<timestamp>
    ```

Next: [Writing Your Own Nodes](writing-your-own-nodes.md).
