# Writing Your Own Nodes

Every A2Bot driver node you'll read about in Part 2 is built from the exact same pieces covered here — a workspace, a package, and a node that publishes, subscribes, or serves. This page still uses only your laptop and turtlesim.

## Concept: workspaces and packages

A **workspace** is a directory tree where you build your own ROS 2 code, kept separate from the system-installed packages under `/opt/ros`. A **package** is the smallest independently-buildable and installable unit inside it — one package might contain one node or several.

!!! laptop "💻 Laptop"
    ```bash
    mkdir -p ~/ros2_ws/src
    cd ~/ros2_ws/src
    ros2 pkg create --build-type ament_python my_first_pkg \
      --dependencies rclpy geometry_msgs
    ```

`--build-type ament_python` matters: it's the same build type every A2Bot package uses (see [Software Architecture](../part2/software-architecture.md)), as opposed to `ament_cmake` for C++ packages.

## Concept: publisher / subscriber

A minimal publisher node — drives the turtle in a circle by publishing `Twist` messages on a timer:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class CircleDriver(Node):
    def __init__(self):
        super().__init__('circle_driver')
        self.pub = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.create_timer(0.1, self.tick)

    def tick(self):
        msg = Twist()
        msg.linear.x = 1.0
        msg.angular.z = 1.0
        self.pub.publish(msg)

def main():
    rclpy.init()
    rclpy.spin(CircleDriver())

if __name__ == '__main__':
    main()
```

Note the pattern: `create_publisher` once in `__init__`, then `.publish()` whenever you have something to send. A subscriber is the mirror image — `create_subscription` takes a callback function that ROS calls automatically whenever a message arrives; you never call it yourself. `a2bot_driver`'s `diff_drive.py` node is exactly this pattern: it subscribes to `/cmd_vel` and publishes to `/wheel_cmd`.

## Concept: service / client

A service **server** defines a callback that receives a request and returns a response. A service **client** calls it and waits (synchronously or via a future) for that response. Unlike a publisher, a client's call fails immediately and clearly if no server exists — the tradeoff for the guaranteed reply.

## Concept: custom messages

The built-in message types (`Twist`, `JointState`, `Odometry`, ...) cover most needs, but a package can define its own in a `.msg` file:

```
# In my_first_pkg/msg/BatteryStatus.msg
float32 voltage
bool charging
```

This requires `ament_cmake` interface-generation support even in a Python package, which is beyond this intro page — the official ROS 2 tutorial on "Creating custom msg and srv files" covers the full setup when you need it.

## Build and run

!!! laptop "💻 Laptop"
    ```bash
    cd ~/ros2_ws
    colcon build --symlink-install
    source install/setup.bash
    ros2 run my_first_pkg circle_driver
    ```

`--symlink-install` links your source files into the install directory instead of copying them, so editing a Python file and re-running takes effect without a full rebuild — used throughout A2Bot development for exactly this reason. Sourcing `install/setup.bash` is a new terminal must-do, exactly like sourcing `/opt/ros/humble/setup.bash` was in [Install & Source](install-and-source.md) — it just adds your workspace's packages on top of the system ones.

## Stale build reflex

If a renamed or moved file's old version seems to keep running despite your source changes, colcon's incremental build state is the usual culprit:

!!! laptop "💻 Laptop"
    ```bash
    rm -rf build install log
    colcon build --symlink-install
    ```

This is the standard "when in doubt" fix throughout the rest of this site, on both the laptop and the Pi.

---

That's the full ROS 2 toolkit this site relies on. From here on, everything is A2Bot-specific — starting with [Electronics & Hardware](../part2/electronics-hardware.md).
