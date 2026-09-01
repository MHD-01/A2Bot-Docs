# Flashing the Arduino Firmware

!!! warning "Firmware source not found in this repository"
    This documentation site was built by reading the actual A2Bot repository, and the Arduino `.ino` firmware source file was **not present anywhere in it** — only the Pi-side driver (`a2bot_driver/serial_bridge.py`) that talks to it. This page documents the serial contract the firmware must implement (confirmed from the Pi-side code) and the general Arduino-on-Ubuntu setup process. If you have the firmware source, flash it with the process below; if you're starting from nothing, it needs to be written to satisfy the protocol described here before `serial_bridge` can talk to it.

## The contract the firmware must satisfy

Confirmed from `a2bot_driver/serial_bridge.py` (see [The Driver: a2bot_driver](../part2/a2bot-driver.md#serial_bridge) for full detail):

- **115200 baud**, connected over USB (appearing to Linux as a CH340-based serial adapter).
- Accepts lines of the form `V<left_rad_s>,<right_rad_s>\n` — target angular velocity for each wheel, in rad/s.
- Sends lines of the form `F<l_pos>,<r_pos>,<l_vel>,<r_vel>\n` continuously, at 50 Hz — encoder position in radians and velocity in rad/s, per wheel.
- Should include its own watchdog: if no `V...` command arrives for some short timeout, it should stop the motors independently, since the Pi-side software watchdog (in `diff_drive`) can't help if the *link itself* has dropped.

## Install the Arduino IDE

!!! laptop "💻 Laptop"
    Download the Arduino IDE (2.x) from arduino.cc, or install via apt:
    ```bash
    sudo apt install -y arduino
    ```

## The CH340 driver

Most Arduino Nano clones use a CH340 USB-to-serial chip rather than the genuine Nano's FTDI chip. Modern Linux kernels include the `ch341` driver already, so no separate driver install is usually needed — but see the brltty conflict below, which is the far more common blocker in practice.

## brltty conflicts with the CH340

Ubuntu's `brltty` service (accessibility support for braille displays) misidentifies CH340 adapters as braille hardware and silently claims the device. Symptom: the Arduino's serial device briefly appears then vanishes, with no error message.

Diagnose:

!!! laptop "💻 Laptop"
    ```bash
    dmesg | grep -i brltty
    ```

    A line like `usb ...: interface 0 claimed by ch341 while 'brltty' sets config #1` confirms it.

Fix:

!!! laptop "💻 Laptop"
    ```bash
    sudo apt remove brltty
    ```

    Then physically unplug and replug the Arduino.

## Flash

1. Connect the Arduino via USB.
2. In the Arduino IDE, select **Tools → Board → Arduino Nano**.
3. Select **Tools → Port** — the port that appeared after removing `brltty`.
4. Open your firmware `.ino` file and click **Upload**.

## Verify over serial

With the firmware flashed and `serial_bridge` **not** running (it would otherwise be the only process allowed to hold the port — see [The Driver: a2bot_driver](../part2/a2bot-driver.md#serial_bridge)):

!!! laptop "💻 Laptop"
    ```bash
    ros2 run a2bot_driver serial_bridge
    ```

    then, in another terminal:

    ```bash
    ros2 topic echo /joint_states
    ```

Position and velocity values changing as you spin a wheel by hand confirms the firmware is correctly reporting `F...` lines.
