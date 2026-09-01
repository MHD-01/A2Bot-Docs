# Discovery & Dashboard

## Concept: finding a robot with no screen

The Pi has no monitor or keyboard attached during normal use — every previous page assumed you already knew its IP address. `a2bot_extras` exists to solve exactly this "how do I find and recover a headless robot" problem, as an **optional** package: nothing in `a2bot_driver`, `a2bot_description`, `a2bot_bringup`, or `a2bot_navigation` depends on it.

## The WiFi provisioning hotspot

If the Pi has no known WiFi network at boot, `wifi_manager` (via the dashboard, below) automatically raises a fallback access point, named and addressed after this robot's own number `X` — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the full per-robot naming convention:

| | |
|---|---|
| Network name | `a2botX-setup` |
| Password | `a2bot123` |
| Address | `10.42.0.X` (put it on a sticker on the chassis) |

!!! pi "🤖 Pi"
    Connect a laptop or phone to `a2botX-setup`, then browse to:
    ```
    http://10.42.0.X:8888
    ```
    This lands on the WiFi setup page — pick a real network and enter its password to get the robot online. Once connected, the hotspot profile is deleted automatically.

Because a single WiFi radio can be a client *or* an access point but never both, connecting the robot to a real network briefly interrupts anyone currently viewing the hotspot page — that's expected.

## nmcli permissions

Joining or hosting a WiFi network via `nmcli` is a privileged NetworkManager operation; without authorization it fails with "not authorized" and (in earlier versions of this code) failed *silently*, making the hotspot look like it simply did nothing. The cleanest fix is a polkit rule granting the `netdev` group those permissions — but on Ubuntu 22.04's polkit 0.105, the modern JavaScript `.rules` format is silently ignored (it needs 0.106+; check with `pkaction --version`), and even a correctly-written older `.pkla`-format rule was found in practice not to cover every `nmcli` operation this code needs (`con add`/`con delete` specifically). The practical fallback actually used here is scoped passwordless sudo for `nmcli` alone:

!!! pi "🤖 Pi"
    ```bash
    echo "$USER ALL=(ALL) NOPASSWD: /usr/bin/nmcli" | sudo tee /etc/sudoers.d/nmcli
    sudo chmod 0440 /etc/sudoers.d/nmcli
    ```

This is a deliberate, acceptable trade-off for a classroom robot holding no sensitive data — not a default recommendation for any machine where that matters.

## The dashboard

Once the robot has network access, a small FastAPI web server shows live telemetry:

!!! pi "🤖 Pi"
    ```bash
    ros2 run a2bot_extras dashboard
    ```

    Browse to `http://<robot-ip>:8888/dashboard` for pose, wheel velocities, which nodes are alive, WiFi status, and a live camera stream (if a camera is running). It also relays start/stop commands to gesture control running on your laptop — see below.

The dashboard runs `ros2 node list` in a subprocess to check which nodes are alive, which means it must explicitly set up the ROS environment (`PATH`, `AMENT_PREFIX_PATH`, etc.) itself — this matters if you ever run the dashboard from `systemd`, since a systemd service does **not** automatically source `~/.bashrc` the way an interactive terminal does.

## Physical recovery buttons

Two hold-to-trigger GPIO buttons give a way to recover the robot without SSH, even if the ROS stack has crashed:

| Action | Trigger |
|---|---|
| Reboot the Pi | Hold button 5 s |
| Forget current WiFi, raise the hotspot | Hold button 5 s |

A 5-second hold (rather than a tap) is deliberate — both actions are disruptive, and the buttons sit on a chassis where they can be brushed by accident. An LED confirms: solid while held, three blinks the instant the hold is accepted, nothing at all if released early. This runs as its own always-on script, independent of the main ROS stack, specifically so it can still recover network access even if ROS never started.

## Gesture control (optional demo)

`gesture_node` runs on your **laptop**, not the Pi — it needs a webcam and more spare compute (for MediaPipe hand-tracking) than a Pi 4 has while also running SLAM and Nav2. It publishes straight to `/cmd_vel`, so **never run it at the same time as autonomous navigation** — they'll fight over the robot.

!!! laptop "💻 Laptop"
    ```bash
    wget -O ~/gesture_recognizer.task \
      https://storage.googleapis.com/mediapipe-models/gesture_recognizer/gesture_recognizer/float16/1/gesture_recognizer.task
    ros2 run a2bot_extras gesture_node
    ```

    Thumb up = forward, peace sign = rotate right, L sign = rotate left, anything else = stop.

!!! warning "numpy version conflict"
    `pip install mediapipe` pulls in NumPy 2.x by default, but Ubuntu 22.04's apt-installed scientific-Python stack (matplotlib, etc.) is compiled against NumPy 1.x. If other tools start breaking after installing MediaPipe, pin it back: `pip install "numpy<2"`.

The dashboard's Start/Stop demo buttons work by relaying to a small launcher service on the laptop, since a browser can't be served a page from the Pi and then make a request straight to `localhost` on the *viewer's* machine:

!!! laptop "💻 Laptop"
    ```bash
    ros2 run a2bot_extras gesture_launcher
    ```

Next: [SLAM & Navigation](slam-and-navigation.md).
