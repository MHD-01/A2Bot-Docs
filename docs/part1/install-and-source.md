# Install & Source

## Concept: what is ROS 2, and why "source" it?

ROS 2 (Robot Operating System 2) is not an operating system — it's a set of libraries, tools, and conventions for writing robot software as small, communicating programs called **nodes**. It installs like any other set of Linux packages, but it doesn't automatically add itself to your shell's environment (`PATH`, Python import paths, etc). Each new terminal has to be told where ROS 2 lives — that's what "sourcing" means: running a script that exports the right environment variables into your current shell session.

This page uses **only your laptop**. No robot is involved yet — everything from here through the rest of Part 1 runs entirely on one machine, using a simulated turtle instead of real hardware, mirroring the official ROS 2 Humble beginner tutorials.

## Install ROS 2 Humble

!!! laptop "💻 Laptop"
    ```bash
    sudo apt update && sudo apt install -y curl gnupg lsb-release
    sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
      -o /usr/share/keyrings/ros-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
      http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
      | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
    sudo apt update
    sudo apt install -y ros-humble-desktop
    ```

`ros-humble-desktop` includes RViz, demos, and simulators — everything Part 1 needs.

## Source it

Sourcing must happen in **every new terminal** before ROS 2 commands work. Add it to your shell startup so it happens automatically:

!!! laptop "💻 Laptop"
    ```bash
    echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```

## Verify

!!! laptop "💻 Laptop"
    ```bash
    ros2 doctor --report
    ```

If this prints a report without complaining that `ros2` is an unknown command, sourcing worked.

## Install turtlesim

The next page uses `turtlesim`, ROS 2's classic teaching simulator — a small window with a turtle you drive around, standing in for a real robot without needing any hardware.

!!! laptop "💻 Laptop"
    ```bash
    sudo apt install -y ros-humble-turtlesim
    ```

Next: [CLI Tools & Core Concepts](cli-tools-core-concepts.md).
