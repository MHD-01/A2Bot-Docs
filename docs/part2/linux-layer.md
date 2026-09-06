# Underneath ROS 2: The Linux Layer

## Concept: ROS 2 is a userspace layer, not an operating system

Every node in this stack is, underneath, an ordinary Linux process — DDS discovery rides on the kernel's UDP multicast, and every piece of real hardware (the Arduino, the lidar, the IMU, the camera) is reached through a standard Linux device file, not anything ROS-specific. The principle from [Software Architecture](software-architecture.md) — "only a ROS 2 node may touch hardware directly" — describes the *ROS graph's* internal discipline. This page is one layer further down: the Linux mechanisms that same handful of nodes actually call into to reach the hardware at all, and that the rest of the stack (the dashboard, `systemd`, WiFi provisioning) depends on outside of ROS entirely.

None of this is taught in depth here — each mechanism already has its own [Toolkit](../toolkit/index.md) reference for the commands. This page's job is narrower: naming each piece and showing exactly where A2Bot's own nodes and services meet it.

## systemd: the outer orchestration layer

ROS 2 launch files compose *nodes*; they know nothing about what starts them or what happens at boot. That's `systemd`'s job, one level up — six services (`a2bot-robot`, `a2bot-dashboard`, `a2bot-rosbridge`, `a2bot-webui`, `a2bot-gpio`, and the `ros2-ready` gate they depend on) start automatically at every boot, one of which — `a2bot-robot` — is itself just `ros2 launch a2bot_bringup robot.launch.py` wrapped in a unit file. See [systemd & Services](../toolkit/systemd-and-services.md) for the full command reference, and [Bringup & Driving](../part3/bringup-and-driving.md#before-you-start-stop-the-background-services) for why these have to be stopped before launching the same stack by hand.

## udev: turning USB chaos into stable names

The Arduino and the lidar are both USB-serial devices, and Linux would otherwise hand them out as `/dev/ttyUSB0`, `/dev/ttyUSB1` — in whichever order they happened to enumerate at boot, which is not guaranteed to be the same twice. `udev` rules keyed on each device's USB vendor:product ID (`1a86:7523` for the Arduino clone's CH340 chip, `10c4:ea60` for the RPLidar's CP2102) create stable symlinks instead: `/dev/arduino` and `/dev/rplidar`. `serial_bridge` and `rplidar_node` open those symlinks, never a raw `ttyUSB*` path. This is also the exact mechanism `brltty` collides with — see [brltty steals the Arduino](../appendices/troubleshooting-index.md#brltty-steals-the-arduino).

## I2C: the bus underneath the IMU

The GY-85's two active chips sit on the Pi's I2C bus (`/dev/i2c-1`), read directly by `a2bot_driver`'s `imu` node — there's no ROS-level I2C abstraction, just the Linux kernel's `i2c-dev` interface. `i2cdetect -y 1` is the standalone way to confirm both chips answer before trusting the node's own logs; see [a2bot_driver — the imu node](a2bot-driver.md#imu) for the addresses and the byte-order quirk between the two chips, and [Electronics & Hardware](electronics-hardware.md) for the wiring.

## Serial: the bus underneath the Arduino

Same category as I2C, one level simpler: a plain UART over USB, at a fixed 115200 baud, reached only through the `/dev/arduino` symlink above. `serial_bridge` is the *only* node allowed to open it — see [a2bot_driver — serial_bridge](a2bot-driver.md#serial_bridge) for the DTR-reset dance this involves and the wire protocol itself.

## libcamera: the Pi's actual camera stack

`camera_ros` (the ROS 2 node `robot.launch.py` starts) is a thin wrapper around **libcamera** — the current Pi camera stack, not the older `picamera`/`raspistill` tooling some tutorials still assume. libcamera is also where the "camera image looks zoomed in" surprise actually comes from: it picks the sensor mode from the requested resolution, and a non-4:3 request on the IMX219 can silently select a cropped, narrower-field-of-view mode instead of the full sensor — see [Dashboard & rosbridge — Troubleshooting](dashboard-and-rosbridge.md#troubleshooting) for the full explanation.

## NetworkManager (`nmcli`): what WiFi provisioning is actually built on

The hotspot-then-join dance covered in [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) and [Discovery & Dashboard](../part3/discovery-and-dashboard.md) is `wifi_manager.py` driving `nmcli` — Ubuntu's standard network configuration tool — not a custom radio driver. See [nmcli & Network Connections](../toolkit/nmcli-and-network-connections.md) for the commands themselves.

## polkit and sudoers: the permission workaround underneath `nmcli`

Creating or joining a WiFi network via `nmcli` is a privileged NetworkManager operation. The clean fix is a `polkit` rule granting the `netdev` group that privilege, but Ubuntu 22.04's `polkit` version doesn't support the modern rule format cleanly for every operation this project needs — so the actual, documented trade-off is scoped passwordless `sudo` for `nmcli` alone. See [nmcli fails with "not authorized"](../appendices/troubleshooting-index.md#nmcli-fails-with-not-authorized) for the full reasoning and the exact commands.

## GPIO: `gpiozero`, and no ROS 2 at all

The physical recovery buttons (`a2bot-gpio.service`, running `gpio_watcher.py`) are worth calling out specifically: unlike every other piece of `a2bot_extras`, this script has **no `rclpy` import anywhere in it**. It's plain Python, `gpiozero`, and `subprocess` calls (`reboot`, and the same `wifi_manager` functions the dashboard uses) — deliberately, so the reboot and forget-WiFi buttons keep working even if the ROS 2 stack never started at all. It uses the `lgpio` pin-factory backend explicitly rather than `gpiozero`'s auto-detection, which was silently picking a broken legacy backend under `systemd` specifically (a `Permission denied: /sys/class/gpio/export` error that never happened running the same script by hand).

## The permission model tying it together

Every subsystem above needs the Pi user in a specific Linux group before it works at all — group membership only takes effect after logging out and back in (or rebooting):

| Group | For |
|---|---|
| `dialout` | Serial (Arduino, RPLidar) |
| `i2c` | IMU |
| `video` | Camera |
| `netdev` | WiFi management (`nmcli`) |

Next: [Software Architecture](software-architecture.md).
