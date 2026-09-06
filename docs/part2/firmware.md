# Firmware: the Arduino's Side

Everything else in Part 2 is a Linux process — this page is the one exception. The Arduino Nano (clone) is a **separate computer entirely**, running its own firmware, reachable only through the serial protocol described in [a2bot_driver — serial_bridge](a2bot-driver.md#serial_bridge). Nothing on the Pi can inspect or debug it directly; the wire protocol is the *only* interface between the two worlds.

## What's confirmed from the Pi side

`serial_bridge` fixes the contract the firmware must implement, at 115200 baud over `/dev/arduino` (see [Underneath ROS 2: The Linux Layer](linux-layer.md#serial-the-bus-underneath-the-arduino) for the udev symlink):

| Direction | Format | Meaning |
|---|---|---|
| Pi → Arduino | `V<left_rad_s>,<right_rad_s>\n` | Target angular velocity for each wheel, rad/s |
| Arduino → Pi | `F<l_pos>,<r_pos>,<l_vel>,<r_vel>\n` | Encoder position (rad) and velocity (rad/s) per wheel, sent continuously at 50 Hz |

This much is verified from the Pi-side code and confirmed working. Beyond it — the actual firmware internals (motor driver PWM mapping, whether encoder velocity is open-loop or closed-loop, whether the firmware has its own command-timeout safeguard) — is the part this page still needs to get right before writing it up.

> **TODO — firmware source location needs confirming before this page goes further.**
> The [Flashing the Arduino Firmware](../appendices/flashing-arduino-firmware.md) appendix states the `.ino` source "was not present anywhere" in the `a2bot_ws` repository. While drafting this page, a `robot_firmware.ino` matching the protocol above **was** found outside that repo, elsewhere on this machine — but its authority as *the* current, deployed firmware hasn't been confirmed, and reading it surfaced a direct conflict with an existing claim in [a2bot_driver — serial_bridge](a2bot-driver.md#serial_bridge): that page currently says the Arduino stops "at its own onboard watchdog" if it stops receiving commands, but the firmware found has **no command-timeout logic anywhere in its main loop** — if serial commands stop arriving for any reason, it keeps driving at the last commanded velocity indefinitely. This needs resolving with the source of truth before either page asserts firmware behavior as fact.

Next: [Part 3 — Setup 1: Raspberry Pi](../part3/setup-1-raspberry-pi.md) or [Setup 2: Laptop](../part3/setup-2-laptop.md).
