# Troubleshooting Index

Symptom-first index of known A2Bot gotchas — each confirmed against this repository, not assumed from general ROS 2 experience.

## brltty steals the Arduino

**Symptom:** `/dev/arduino` briefly exists after plugging in, then vanishes with no error. `serial_bridge` fails to connect or connects then immediately loses the port.

**Cause:** Ubuntu's `brltty` service (braille display accessibility support) misidentifies CH340 USB-serial adapters — used by most Arduino Nano clones — as braille hardware, and silently claims the device a few seconds after it appears.

**Diagnose:**

!!! pi "🤖 Pi"
    ```bash
    dmesg | grep -i brltty
    ```

    A line like `interface 0 claimed by ch341 while 'brltty' sets config #1` confirms it.

**Fix:**

!!! pi "🤖 Pi"
    ```bash
    sudo apt remove brltty
    ```

    Then unplug and replug the Arduino.

## SSH connection troubleshooting

**Symptom:** `ssh` times out, says the host is not found, asks for a password and then rejects it, or fails before the terminal opens.

**Common causes:**

- The robot and laptop are not on the same network.
- The hostname or robot number is wrong.
- The Ethernet cable is not connected properly or the link is not live.
- The robot's static Ethernet link is not configured on the same subnet.
- The SSH password is correct but the username in the command is misspelled.

**Checks to do first:**

1. Confirm both devices are on the same network.
   - For WiFi, follow the dashboard and network setup flow in [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md).
   - For Ethernet, follow [IP Addresses & Your Network — Direct Ethernet link: a fixed private network](../toolkit/ip-addresses-and-your-network.md#direct-ethernet-link-a-fixed-private-network).
2. Check the exact SSH command and robot number.
   ```bash
   ssh a2botX@a2botX-host.local
   ```
   Replace `X` with the robot's actual number, and check that the username and hostname match exactly.
3. If hostname resolution fails, connect by IP instead of `.local`.
   ```bash
   ssh a2botX@10.42.0.X
   ```
   or, for the direct Ethernet link:
   ```bash
   ssh a2botX@10.0.0.10X
   ```
4. Make sure the Ethernet cable is seated correctly and the link is active.
   ```bash
   nmcli device status
   ip a
   ```
5. If the password prompt appears and the password is rejected even though it seems correct, re-check the username before assuming the password is wrong. A common mistake is typing the wrong robot number or misspelling the username (`a2bot5` instead of the correct value).

**If you are using the direct Ethernet setup:** the robot should be configured with a fixed address on the same private subnet as the laptop. See [IP Addresses & Your Network — Direct Ethernet link: a fixed private network](../toolkit/ip-addresses-and-your-network.md#direct-ethernet-link-a-fixed-private-network) for the exact `nmcli`/`netplan` procedure and the address conventions used in this project.

**Fix:** once the network, cable, hostname, and username are all correct, reconnect and complete the SSH host confirmation prompt with `yes` when prompted.

## The GY-85 reads plausible garbage instead of erroring

**Symptom:** IMU data looks like *numbers*, not obviously wrong, but the EKF's heading drifts strangely or the values don't match how you're actually moving the robot.

**Cause:** The GY-85 is three separate chips at three I2C addresses (ADXL345 accelerometer at `0x53`, ITG-3200 gyroscope at `0x68`/`0x69`, an unused magnetometer), and critically, **the two chips in use have opposite byte order** — the accelerometer is little-endian, the gyroscope is big-endian, on the same board. Code written against a single-chip IMU won't crash against this; it will read garbage that still looks like plausible numbers.

**Diagnose:** `a2bot_driver`'s `imu` node checks both chip IDs at startup and logs a warning if either is wrong — check its startup log. Manually:

!!! pi "🤖 Pi"
    ```bash
    sudo i2cdetect -y 1     # expect devices at 0x53 and 0x68
    ```

    ADXL345 `DEVID` register (`0x00`) should read `0xe5`; ITG-3200 `WHO_AM_I` register (`0x00`) should read `0x68` or `0x69`. See [a2bot_driver](../part2/a2bot-driver.md#imu) and [Electronics & Hardware](../part2/electronics-hardware.md) for full register detail.

## The EKF silently outputs nothing

**Symptom:** `/odometry/filtered` never publishes, and there's no error message anywhere.

**Cause:** `robot_localization`'s EKF needs **yaw and velocity**, not just x/y position, from at least one input to fully initialize — a config that takes only position from wheel odometry looks reasonable but never produces output.

**Diagnose:**

!!! pi "🤖 Pi"
    ```bash
    ros2 topic hz /odometry/filtered
    ```

    Zero Hz confirms the filter is stuck. A2Bot's shipped `ekf.yaml` is correctly configured (it does include yaw and velocity) — see [Sensor Fusion / EKF](../part2/sensor-fusion-ekf.md) for the exact fields and a note about a misleading comment in that same file.

## nmcli fails with "not authorized"

**Symptom:** The WiFi hotspot or network-join fails, sometimes silently.

**Cause:** Creating a hotspot or joining a network via `nmcli` is a privileged NetworkManager operation. On Ubuntu 22.04's polkit 0.105, the modern `.rules` (JavaScript) format needed for a clean fix is silently ignored (needs 0.106+), and even the older `.pkla` format was found not to cover every operation needed (`con add`/`con delete` specifically).

**Fix (the trade-off actually used in this project):**

!!! pi "🤖 Pi"
    ```bash
    echo "$USER ALL=(ALL) NOPASSWD: /usr/bin/nmcli" | sudo tee /etc/sudoers.d/nmcli
    sudo chmod 0440 /etc/sudoers.d/nmcli
    ```

    Acceptable for a classroom robot holding no sensitive data — not a general recommendation. See [Discovery & Dashboard](../part3/discovery-and-dashboard.md).

## RViz shows no robot model

**Symptom:** Topics and `tf` all look correct, but RViz shows an outline, or nothing, where the robot should be — even though everything is working over the network from a remote machine.

**Cause:** RViz resolves the URDF's `package://a2bot_description/meshes/...` mesh references from the **local** filesystem via the ROS package index — it never fetches meshes over the network. A machine that has never built `a2bot_description` locally shows this exact confusing failure, because everything else (topics, tf) still works fine.

**Fix:** Build the workspace locally on whichever machine is running RViz — see [Setup 2 — Laptop, step 3](../part3/setup-2-laptop.md#3-build-the-workspace-locally) (or [Setup 1 — Raspberry Pi, step 12](../part3/setup-1-raspberry-pi.md#12-clone-and-build-the-a2bot-workspace) if it's the Pi itself running RViz).

## ROS 2 topics work over ping/SSH but discovery fails

**Symptom:** `ping` and `ssh` between the Pi and laptop both work fine, but `ros2 topic list` on one machine never shows the other's topics.

**Cause:** AP/client isolation, common on shared or lab WiFi (and even some home routers), blocks device-to-device traffic on the same network while still allowing normal internet access. ROS 2 discovery depends on UDP multicast, which isolation blocks — with no error, since ping and SSH use different traffic paths that isolation doesn't touch.

**Fix:** No software-side fix exists. Disable AP/client isolation on the router, or bypass it entirely with a direct Ethernet link — see below.

## Static Ethernet Link Setup

For the direct Ethernet static-IP tutorial, see [IP Addresses & Your Network — Direct Ethernet link: a fixed private network](../toolkit/ip-addresses-and-your-network.md#direct-ethernet-link-a-fixed-private-network). That section is now the canonical learning reference for the fixed `10.0.0.0/24` setup used in this project.

## Stale build — code changes don't seem to take effect

**Symptom:** You renamed or moved a file, rebuilt, but the old behavior persists.

**Fix (the standard reflex, on either machine):**

!!! both "🔗 Both"
    ```bash
    rm -rf build install log
    colcon build --symlink-install
    ```

## Undervoltage on the Pi

**Symptom:** Flaky WiFi, USB devices dropping out, unexplained reboots.

**Diagnose:**

!!! pi "🤖 Pi"
    ```bash
    dmesg | grep -i voltage
    ```

    Any "under-voltage detected" line means the power supply is the real culprit. A2Bot needs 5V/3A minimum.

## A rename leaves stale files/folders that a text sweep can't fix

**Symptom:** After renaming a ROS2 package project-wide (e.g. `old_name_driver` → `new_name_driver`), one of three things happens depending on which piece was missed:

- `colcon build` fails with `can't copy '.../resource/<new_name>': doesn't exist or not a regular file`.
- The package builds, but a built executable fails at launch with `package '<new_name>' found... but libexec directory '.../lib/<OLD_name>' does not exist` — note the message names the *new* package but the *old* path.
- A launch file fails with a plain `No such file or directory` for a filename that still exists on disk under its old name.

**Cause:** A text-content search-and-replace across `.py`/`.xml` files only rewrites what's *inside* files. Three specific things are identified by their filename itself, not their contents, so a content sweep is structurally incapable of catching them:

1. **The ament resource marker** — an empty file at `<package>/resource/<package_name>`. Its entire job is to exist under that exact name; renaming an empty file's contents does nothing, since it has none. The name itself has to be renamed (`mv`).
2. **`setup.cfg`**, sitting next to `setup.py`, commonly hardcodes the package name as a literal string in `script_dir`/`install_scripts` — completely independent of `setup.py`'s `package_name` variable. A sweep that only checks `setup.py` will miss this, and the mismatch is silent until something tries to launch the installed executable.
3. **Any non-Python filename referenced by path rather than by import** — e.g. a URDF file opened by a launch file's file-open call. Renaming the launch file's *reference* to a new filename does not rename the actual file still sitting on disk under the old name.

**Verified live in this repo:** items 1 and 3 are currently handled correctly — every package's `resource/<package_name>` marker matches its current name, and `a2bot_description/urdf/a2bot.urdf` matches what `description.launch.py`/`sim.launch.py` actually open. But item 2 is a **real, currently-present bug**, caught while writing this entry, not a hypothetical:

```text
a2bot_description/setup.py   -> package_name = 'a2bot_description'
a2bot_description/setup.cfg  -> script_dir=$base/lib/A2Bot_description   (wrong case)

a2bot_bringup/setup.py       -> package_name = 'a2bot_bringup'
a2bot_bringup/setup.cfg      -> script_dir=$base/lib/A2Bot_bringup       (wrong case)
```

Both `setup.cfg` files still read the capitalized `A2Bot_*` form from before this project's naming was fully lowercased. Linux paths are case-sensitive, so this is exactly the silent-mismatch failure mode described above, just waiting for whichever of these two packages gets built and launched first.

**Fix:**

!!! both "🔗 Both"
    ```bash
    sed -i 's/A2Bot_description/a2bot_description/g' a2bot_ws/src/a2bot_description/setup.cfg
    sed -i 's/A2Bot_bringup/a2bot_bringup/g' a2bot_ws/src/a2bot_bringup/setup.cfg
    ```

    Then rebuild both packages (`colcon build --symlink-install --packages-select a2bot_description a2bot_bringup`) and confirm the installed executables land under `install/<package>/lib/<package>/`, not a differently-cased directory.

## A systemd service comes up "active" on the wrong ROS_DOMAIN_ID / RMW_IMPLEMENTATION

**The risk:** `systemctl status a2bot-robot` reporting `active (running)`, with real processes alive and using real CPU, is not proof the service is on the right ROS graph. This project's setup pages ([Setup 1](../part3/setup-1-raspberry-pi.md#7-set-the-ros-domain-id), [Setup 2](../part3/setup-2-laptop.md#4-match-the-ros-domain-id)) have a human export `ROS_DOMAIN_ID` and `RMW_IMPLEMENTATION` from `~/.bashrc`. A systemd unit that starts its process via a login shell (`ExecStart=/bin/bash -l -c '...'`) runs a **login** shell, but not an **interactive** one — and `.bashrc`'s early-return guard for non-interactive shells means those `export` lines can be silently skipped, leaving a service alive but silently on the ROS 2 defaults (domain `0`, typically FastDDS) — invisible to `ros2 node list`/`ros2 topic list` from a correctly-configured terminal, which looks identical to "nothing is running."

**Verified in `services/services_files/`: this project already learned this lesson and closed it.** Every service that actually touches ROS2 — `a2bot-dashboard.service`, `a2bot-gpio.service`, `a2bot-robot.service`, and `a2bot-rosbridge.service` — sets both variables explicitly in `[Service]`, independent of `.bashrc`, using **this robot's own domain ID** rather than a value shared across the fleet:

```ini
Environment=ROS_DOMAIN_ID=X
Environment=RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

(`X` = this robot's own number, matching its hostname and hotspot — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md). A unit file with a domain ID that doesn't match this robot's own number is exactly the kind of mismatch the diagnostic below exists to catch.)

(`a2bot-webui.service` is the one exception, and correctly so — it only runs `npm run dev`, never calls the `ros2` CLI, so it has no need for either variable.)

Worth noting: `a2bot-rosbridge.service`'s own comment justifies its `-l` flag by claiming a login shell alone sources `.bashrc` and provides `ROS_DOMAIN_ID` that way — which is the exact mechanism this entry is skeptical of (a non-interactive `-c` invocation typically still hits `.bashrc`'s interactive-only guard). It doesn't matter in practice here: the explicit `Environment=` line is set on that same service regardless, so it's correct either way `-l` actually behaves.

**If you add a new ROS-touching service, don't skip this** — diagnose it the same way this project would have caught it originally, by checking the process's *actual* environment rather than trusting `systemctl status`:

!!! pi "🤖 Pi"
    ```bash
    sudo systemctl show <service> -p MainPID
    cat /proc/<that PID>/environ | tr '\0' '\n' | grep -E "ROS_DOMAIN_ID|RMW_IMPLEMENTATION"
    ```

    Empty output, or values that don't match this robot's number, means the new unit needs the same `Environment=` lines shown above added to its `[Service]` section.

## Prefer an active readiness check over a fixed `sleep` for service startup ordering

Not a bug — a pattern worth understanding, since this project actually uses **both** approaches side by side for different reasons, verified in `services/services_files/`. `Requires=`/`After=` in a systemd unit only guarantee that one service's *start attempt* happens after another's — not that the dependency is actually *ready* to serve requests by the time the second service starts.

**Where a fixed `sleep` is still used, deliberately:**

- `a2bot-robot.service` — `ExecStartPre=/bin/sleep 20`, because USB/I2C device enumeration after boot has no clean endpoint to poll; its own comment notes an earlier `5s` value wasn't enough and caused the lidar node to crash on start. 20s was arrived at empirically, not computed.
- `a2bot-rosbridge.service` — `ExecStartPre=/bin/sleep 10`, a deliberate fixed gap after `a2bot-webui` is told to start, before rosbridge comes up.

**Where the poll-based readiness gate is actually used:** exactly one place — `a2bot-webui.service`, waiting on `a2bot-dashboard`. Its own comment states the reasoning directly: `Requires=` only proves the dashboard *service* started, not that it's *answering requests* yet — "the exact 'active does not mean working' gap this project hit repeatedly with the ROS services." Its `ExecStartPre` polls the dashboard's own `/api/status` endpoint, which only responds once FastAPI is genuinely serving, retrying for up to 60 seconds before giving up:

```bash
ExecStartPre=/bin/bash -c '\
  for i in $(seq 1 30); do \
    curl -sf http://localhost:8888/api/status > /dev/null 2>&1 && exit 0; \
    sleep 2; \
  done; \
  echo "Dashboard did not become ready in time"; exit 1'
```

**The distinction that decides which pattern fits:** a fixed `sleep` is the right call when there's no cheap, meaningful signal to poll for (hardware enumeration timing, an arbitrary "give it a moment" gap) — polling doesn't help if there's nothing to ask. The readiness-poll pattern is worth the extra script only when the dependency exposes something you can actually check (here, an HTTP endpoint that only responds once genuinely up). If you're adding a new service that depends on one with a real health/status endpoint, follow `a2bot-webui.service`'s pattern above rather than guessing a sleep duration.

## The WiFi radio can't scan while the setup hotspot is up

**Symptom:** The WiFi setup page's network dropdown shows only the hotspot's own network, or an empty list, even though real networks are nearby — or, if a naive implementation scanned by dropping the hotspot, the very device viewing the setup page loses its connection mid-scan.

**Cause:** The Pi has a single WiFi radio, which can be an access point (broadcasting `a2botX-setup`) or a client scanning for networks — never both at the same instant. Scanning requires the radio to leave AP mode.

**Verified in `wifi_manager.py`:** this is already handled with a pre-capture cache, not a live scan on every page load:

- `_start_hotspot()` runs a real scan (`_raw_scan()`) **before** raising the access point, while the radio is still in client mode, and stores the result in module-level `_cached_networks` / `_cache_time`.
- `scan_networks(force=False)` — what a routine page load or reload calls — just returns that cache. It never touches the radio, so it can't disconnect anyone or return an empty "just the hotspot" list.
- `scan_networks(force=True)` — the dashboard's explicit **Rescan** action — is the only path that actually drops the hotspot, re-scans, and restores it afterward (`wifi.html`'s `rescan()` shows a `confirm()` warning about the ~10s disconnection before calling it, for exactly this reason).

No fix needed here — this entry exists so the cache/force split isn't mistaken for a bug the next time someone reads the code looking for "why doesn't this just scan live."

## A hold-to-confirm GPIO button could re-fire while still held

**Symptom (the failure mode this guards against):** A hold-to-confirm loop that just checks "has N seconds elapsed" on every poll will keep re-satisfying that condition for as long as the button stays down past the threshold — a human does not release at the exact instant the threshold is hit. Without a guard, this could re-run the action (a second `reboot`, another `forget_current_network()` call) multiple times during one long hold.

**Verified in `gpio_watcher.py`'s `watch_button()`:** this is already guarded against with an explicit triggered-state flag, not a bare threshold check:

- A `triggered` boolean starts `False` and is set `True` the instant the hold is accepted.
- The action only ever fires inside `if not triggered and held >= HOLD_SECONDS:` — once `triggered` flips, that branch can't run again for the rest of this hold.
- Deliberately no `return` after the action fires: the code falls through with the LED turned off and keeps polling `while button.is_pressed`, rather than looping back into the threshold check — so a long or stuck hold sits idle instead of re-arming.
- Two visually distinct LED signals mark two different events, not the same action twice: **3 blinks** the instant the hold is accepted (action fired), and a separate **1 blink** only once the button is actually released (confirms release, not a second trigger).

## Not covered here

A systemd-service-plus-Node.js PATH interaction (where `nvm`'s `.bashrc` fix doesn't apply to non-interactive shells) is a known general gotcha in projects that run a Node.js service under systemd. This repository's dashboard and gesture services are Python (FastAPI/Flask), not Node — no systemd-managed Node.js service was found here, so this gotcha is **not** documented as applying to A2Bot. If a Node-based service is added later, revisit this.
