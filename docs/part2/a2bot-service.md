# Closed-Loop Motion: a2bot_service

`a2bot_service` is one node, `service_server`, that turns `/cmd_vel` + `/odom` into four request/response services for point-to-point motion — plus three thin CLI clients for calling them from a terminal. It's already part of the standard robot bringup, not a separate thing you opt into.

## Concept: closing the loop against odometry

Every driving method covered so far — [teleop keyboard](../part3/bringup-and-driving.md#drive-it-manually), gesture control (see [Discovery & Dashboard](../part3/discovery-and-dashboard.md#gesture-control-optional-demo)) — is **open-loop**: it publishes a `/cmd_vel` rate command and leaves it to a human to watch the robot and decide when to stop. `a2bot_service` closes that loop: a caller asks for "move 1 m" or "turn 90°" as a single request, and gets back a single response only once the robot's own odometry confirms the target was reached (or the attempt failed) — no manual timing required.

## The four services

| Service | Type | Request | Response |
|---|---|---|---|
| `/a2bot/move_distance` | `a2bot_service/MoveDistance` | `distance` (m), `speed` (m/s, `0` = use default) | `success`, `message` |
| `/a2bot/turn_angle` | `a2bot_service/TurnAngle` | `angle` (deg), `speed` (rad/s, `0` = use default) | `success`, `message` |
| `/a2bot/return_home` | `std_srvs/Trigger` | — | `success`, `message` |
| `/a2bot/stop` | `std_srvs/Trigger` | — | `success`, `message` |

`move_distance` and `turn_angle` needed their own custom `.srv` types (a distance/angle plus a speed); `return_home` and `stop` take no arguments, so they reuse the standard `std_srvs/Trigger` rather than defining an empty custom type for no reason.

## How a move or turn actually runs

Each call runs a simple loop: publish a constant `/cmd_vel` command, then every 50 ms check the distance travelled (or heading turned) against the request's target using the latest `/odom` pose. It stops and reports success once within tolerance (0.03 m / 0.05 rad by default), or reports failure once a deadline — computed from distance ÷ speed, capped at `max_duration` (60 s) — passes first.

Because that loop **blocks the callback for the whole maneuver**, `service_server` runs a `MultiThreadedExecutor` with a `ReentrantCallbackGroup` rather than the single-threaded default used elsewhere in this stack. Without that, `/a2bot/stop` could never actually interrupt a move already in progress — it would just queue behind it. With it, a `stop` call runs on its own thread, flips a `stop_requested` flag, and the in-progress move/turn loop notices it on its next 50 ms check and halts immediately.

## `return_home`: undo by replay, not navigation {#return_home}

There's no "drive back to a remembered pose" logic here. Instead, every `move_distance`/`turn_angle` call that got underway is pushed onto an in-memory stack (`command_history`). `return_home` pops that stack most-recent-first and reissues each entry **negated** — e.g. "forward 1 m, turn left 90°" undoes as "turn right 90°, backward 1 m". This sidesteps needing a stored absolute position to navigate back to; it only needs to remember what it did.

!!! warning "return_home can overshoot after an interrupted move"
    The history entry records the **requested** distance/angle, not how far the robot actually got — deliberately, so that even a call which later timed out or was `/stop`ped mid-way still gets *some* reversal instead of being silently dropped (see the comments above `_move_cb`/`_turn_cb` in `service_server.py`). But this means a move that, say, timed out 0.3 m into a requested 1.0 m still gets undone as a full 1.0 m reversal — overshooting the real starting point by 0.7 m. This only bites on a call that didn't cleanly succeed; a normal completed move/turn reverses accurately, since the tolerance keeps the requested and actual amounts effectively equal. Worth a human fix (record actual travelled distance, not the target) before relying on `return_home` after a `stop` or timeout.

`command_history` is also **not thread-safe** — it's a plain list mutated from whichever callback thread happens to be running. This has never mattered in practice because the dashboard and CLI clients only ever issue one movement command at a time, but two overlapping move/turn/return_home calls from different clients at once is a real (currently unhandled) race.

## Runs against raw `/odom`, not the fused estimate

`odom_topic` defaults to `/odom` — `a2bot_driver`'s raw wheel odometry, not `/odometry/filtered` from the [EKF](sensor-fusion-ekf.md). For a single short move or turn this rarely matters (wheel drift needs distance/time to accumulate), but it's the same drift-prone source the EKF exists to correct, so it's worth knowing this package isn't using it. Nothing currently overrides this at launch; pass `odom_topic:=/odometry/filtered` as a parameter if you want closed-loop moves checked against the fused pose instead.

## Already wired into the stack

`a2bot_bringup/launch/robot.launch.py` starts `service_server` (as node `command_services`) alongside the driver chain, `robot_state_publisher`, the lidar, and the camera — with no parameters overridden, so it runs with the defaults above (`invert_cmd_vel: true` included). You don't need to launch it separately.

The [dashboard](../part3/discovery-and-dashboard.md#the-dashboard)'s move/turn/return-home/stop controls aren't a separate implementation — its `/api/move-distance`, `/api/turn-angle`, `/api/return-home`, and `/api/stop-motion` endpoints are thin wrappers that call these same four services from their own ROS node inside `a2bot_extras/dashboard.py`.

## The CLI clients

For calling the services directly from a terminal, without the dashboard:

!!! pi "🤖 Pi"
    ```bash
    ros2 run a2bot_service move_client 1.0 --speed 0.2      # forward 1 m
    ros2 run a2bot_service turn_client 90 --speed 0.5        # turn left 90°
    ros2 run a2bot_service return_home_client                # undo everything above
    ```

Each prints the service's `message` and exits `0` on success, `1` on failure or if the service isn't reachable within 5 s — safe to use in a shell script's exit-code check. There's no CLI client for `/a2bot/stop`; call it directly:

!!! pi "🤖 Pi"
    ```bash
    ros2 service call /a2bot/stop std_srvs/srv/Trigger
    ```

Next: [Part 3 — Setup 1: Raspberry Pi](../part3/setup-1-raspberry-pi.md) or [Setup 2: Laptop](../part3/setup-2-laptop.md).
