# rqt

## What this is, and why it matters

`rqt` is a GUI framework that wraps several ROS 2 introspection tools as plugins in one window. This course uses two of them: **rqt_graph**, which draws the actual node/topic wiring of a running system as a diagram, and **rqt_plot**, which live-plots a single numeric value over time. Both answer questions that are slow or awkward to answer by reading `ros2 topic echo` output scrolling past.

## rqt_graph: seeing the wiring, not just the list

!!! laptop "💻 Laptop"
    ```bash
    ros2 run rqt_graph rqt_graph
    ```

The diagram uses a small, fixed set of shapes and arrows:

- **Ellipses are nodes** — running programs, like `/odometry` or `/diff_drive`.
- **Rectangles are topics** — named channels, like `/odom` or `/cmd_vel`.
- An arrow **from a node into a topic** means that node **publishes** to it.
- An arrow **from a topic into a node** means that node **subscribes** to it.

The one rule worth internalizing: **two nodes never draw a direct arrow to each other.** Even when a chain looks like a straight line from one node to another, it always passes through a topic (rectangle) in between — ROS 2 nodes only ever communicate by publishing to, or subscribing from, a topic, never node-to-node directly. If the graph ever appears to show a node connected straight to another node with no topic between them, that's a rendering/layout quirk, not a real direct connection — one doesn't exist.

## Why this is faster than reading logs

If a node is supposed to be receiving something and isn't, the fastest check isn't scrolling logs — it's opening `rqt_graph` and looking for the arrow. An expected subscriber with no incoming arrow from the topic it should be reading, or a publisher with no outgoing arrow to where it should be sending, is visible instantly, without needing to know what a working log even looks like. It won't tell you *why* a connection is missing — a topic name typo, a node that crashed on startup, a launch file that never started it — but it tells you *where* to look next, immediately.

## rqt_plot: watching a number change live

Same framework, a different plugin — instead of a wiring diagram, `rqt_plot` graphs one numeric field from a topic over time, updating live:

!!! laptop "💻 Laptop"
    ```bash
    ros2 run rqt_plot rqt_plot /odom/twist/twist/linear/x
    ```

This is the same underlying data `ros2 topic echo` would print as scrolling numbers, just easier to actually watch — useful for confirming a value is smoothly changing (or stuck at zero) without reading a wall of text.

## Where this shows up in the course

Both tools are the practical way to sanity-check that [Bringup & Driving](../part3/bringup-and-driving.md)'s launch actually wired everything the way it's supposed to, and to visually confirm the driver chain described in [ROS2 on Real Hardware](../part3/ros2-on-real-hardware.md) is really connected end to end before assuming a deeper problem.
