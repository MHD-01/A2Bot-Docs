# Dashboard & rosbridge: a2bot_extras

`a2bot_extras` also provides WiFi provisioning, the GPIO recovery buttons, and gesture control — all covered in [Discovery & Dashboard](../part3/discovery-and-dashboard.md). This page covers a different slice of the same package: the browser-based **control interface**, and **rosbridge**, the mechanism that lets any browser reach the ROS 2 graph at all without being a ROS 2 node itself.

## Two separate web interfaces — do not conflate them

This trips up everyone who works on this robot, so it's worth stating plainly before anything else: **"the dashboard" is two different programs, on two different ports**, with two different ways of reaching ROS.

| | Control interface | Setup/telemetry page |
|---|---|---|
| systemd unit | `a2bot-webui.service` | `a2bot-dashboard.service` |
| Port | **7777** | **8888** |
| What it is | React single-page app | Python FastAPI server that is *also* a ROS 2 node |
| Talks to ROS via | **rosbridge WebSocket** (port 9090) | **`rclpy` directly** — it's a real node |
| Source | `a2bot_extras/a2bot_extras/Turtlebot interface/robot-control-interface/` | `a2bot_extras/a2bot_extras/dashboard.py` |
| Pages | Home / Control / Monitor / Circuit tabs | `/` WiFi setup, `/dashboard` telemetry |

The FastAPI page is what [Discovery & Dashboard](../part3/discovery-and-dashboard.md) is about — it raises the WiFi provisioning hotspot, and it's what a robot with no known network can still be reached through. The React control interface, covered here, is the other one: it never touches hardware or `rclpy` itself, and reaches the robot purely as a rosbridge client, exactly like any other browser could. It does call two of the FastAPI page's HTTP endpoints directly (see [Subscribing, publishing, and calling services](#subscribing-publishing-and-calling-services)) — that's the only place the two interfaces actually touch.

## How the control interface was built

Stack:

- **React 19** + **Vite 7** (build tool and dev server)
- **Tailwind CSS 3** + **daisyUI 5** for styling/components
- **roslib** (roslibjs) 1.4.1 — the ROS 2 client library for JavaScript
- **lucide-react** for icons, **nipplejs** for the on-screen joystick

Layout under `robot-control-interface/src/`:

- `App.jsx` — the app shell: navbar, tab bar, routing between pages
- `components/pages/` — `HomePage`, `ControlPage`, `MonitorPage`, `CircuitPage`, `ConnectGate` (a `SettingsPage` exists but its tab is currently commented out)
- `components/` — reusable widgets: `CameraFeed`, `LidarVisualization`, `Map2D`, `JoystickController`, `DistanceControl`, `RobotStatus`, `NodesTopicsPanel`, `ServicesStatusPanel`, `WifiSettings`
- `context/` — `ROSProvider.jsx`, `ROSContext.js`, `useROS.js`
- `lib/robotConfig.js` — connection settings and topic names

**The one architectural decision worth understanding:** every ROS interaction lives in a single React context, `ROSProvider`. Components never open their own connection — they call `useROS()` and get the one shared connection. The whole app therefore has **exactly one WebSocket** to rosbridge, and one place where subscriptions are created and torn down. A connection per component would mean every tab switch churns subscriptions — a cost this Pi can't spare.

**How it's served:** `a2bot-webui.service` runs `npm run dev` — the Vite dev server, on port **7777**, with `host: true` (listen on all interfaces, so other machines on the LAN can reach it) and `strictPort: true` (fail loudly instead of silently drifting to another port and breaking the link the setup page uses to reach it).

!!! note "A dev server in production is a known shortcut"
    `npm run dev` is a development server, not meant for permanent use. `npm run build` plus a static file server is the correct long-term setup — worth revisiting once the interface stabilizes.

## Concept: why browsers need rosbridge at all

ROS 2 nodes talk to each other over **DDS**, which relies on UDP multicast and shared memory. A web browser cannot speak DDS — it has no access to raw sockets, and the browser security model forbids it outright. A browser can never be a ROS 2 node.

**rosbridge** is what closes that gap. It's a ROS 2 node that:

1. Joins the ROS 2 graph normally — it *is* a real DDS participant.
2. Opens a **WebSocket server on port 9090**, which browsers *can* connect to.
3. Translates between the two: **JSON/CBOR messages over the WebSocket** on one side, real ROS 2 pub/sub and service calls on the other.

The browser itself never joins the graph. It sends rosbridge a JSON message meaning "subscribe me to `/scan`," and rosbridge — which is in the graph — subscribes on its own behalf and forwards each message back over the WebSocket.

Three packages are involved, installed together via apt as `ros-humble-rosbridge-suite`:

- **`rosbridge_server`** — the WebSocket server itself.
- **`rosbridge_library`** — the JSON ↔ ROS message conversion.
- **`rosapi`** — a companion node exposing **graph introspection** as ROS services (`/rosapi/nodes`, `/rosapi/topics`, and others). This is what makes a "list all nodes and topics" feature in a browser possible at all — without `rosapi`, a browser client can publish and subscribe, but it has no way to ask *what exists*. It starts automatically as part of the standard rosbridge launch file.

On A2Bot, `a2bot-rosbridge.service` starts it with:

```bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```

## The data flow

```
 Browser (React + roslibjs)             FastAPI dashboard (dashboard.py)
         │                                        │
         │  ws://<host>:9090                      │  in-process
         │  JSON / CBOR                            │  (it IS a ROS 2 node)
         ▼                                        ▼
 ┌──────────────────────┐              ┌───────────────────────┐
 │   rosbridge_server     │◄─── DDS ───►│      ROS 2 graph         │
 │  + rosbridge_library    │            │  (topics and services)   │
 │  + rosapi                │◄─── DDS ───┤                          │
 └──────────────────────┘              └───────────────────────┘
```

The React app and the FastAPI page reach the exact same graph through two structurally different paths — one hops through a WebSocket and a translation layer, the other is a normal in-process ROS 2 node. Neither one talks to hardware directly; both go through the graph, same as everything else in this project.

## Service configuration and startup order

`a2bot-rosbridge.service` sets several things that are easy to miss and painful to debug if they're wrong:

```ini
WorkingDirectory=/home/a2bot2
Environment=HOME=/home/a2bot2
Environment=ROS_DOMAIN_ID=X
Environment=RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ExecStartPre=/bin/sleep 10
ExecStart=/bin/bash -l -c 'source /opt/ros/humble/setup.bash && \
    source /home/a2bot2/a2bot/a2bot_ws/install/setup.bash && \
    ros2 launch rosbridge_server rosbridge_websocket_launch.xml'
```

(`X` is this robot's own number, matching its hostname and hotspot — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md).)

Each line here came from a real failure, not from following a template:

- **`HOME` and `WorkingDirectory`** — systemd services get *no* home directory by default, unlike a login shell. Without these, ROS 2's CLI tools try to write to `~/.ros` and fail silently. This was the actual cause of recurring `ros2 node list timed out after 5 seconds` errors.
- **`ROS_DOMAIN_ID=X` and `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`** — *every* node in the system must agree on both. A node on a different domain ID or a different RMW is invisible to the rest of the graph while still showing `active (running)` in `systemctl status` — one of the most confusing ROS 2 failure modes there is. See [A systemd service comes up "active" on the wrong ROS_DOMAIN_ID / RMW_IMPLEMENTATION](../appendices/troubleshooting-index.md#a-systemd-service-comes-up-active-on-the-wrong-ros_domain_id-rmw_implementation) for a fuller account.
- **`bash -l`** — a login shell, so `.bashrc` (where these same variables also live for interactive use) is actually read.
- **`ExecStartPre=/bin/sleep 10`** — a deliberate stagger so several ROS processes don't all hit the ROS 2 daemon at the exact same instant during boot.

Boot order across the four units: `ros2-ready.service` is required by all of them. `a2bot-webui.service` goes one step further for `a2bot-dashboard.service` specifically — it doesn't just wait for systemd to have *started* it, its `ExecStartPre` polls `http://localhost:8888/api/status` up to 30 times, 2 seconds apart, until the dashboard genuinely answers HTTP. **"systemd says active" and "the service actually works" are different claims** — see [systemd & Services](../toolkit/systemd-and-services.md#what-active-actually-proves-and-what-it-doesnt) for why that gap exists in general, and [Prefer an active readiness check over a fixed sleep](../appendices/troubleshooting-index.md#prefer-an-active-readiness-check-over-a-fixed-sleep-for-service-startup-ordering) for why this particular wait is a poll rather than another fixed sleep.

## How the connection is opened

All of this lives in `context/ROSProvider.jsx`.

**Working out the address.** `lib/robotConfig.js` builds the WebSocket URL as `ws://<host>:9090`. `host` isn't hardcoded — a helper called `liveHost()` reads `window.location.hostname`, i.e. whatever address the browser used to load the page in the first place. Since the page is served by the robot itself, that address is by definition a working route back to it. The payoff: the robot's IP never has to be edited by hand as it moves between networks — load the dashboard from wherever it's currently reachable, and rosbridge is targeted correctly automatically. It only falls back to a saved or default IP when the page is served from `localhost` — i.e. during development against a remote robot.

**Opening the connection:**

```js
const ros = new ROSLIB.Ros({ url: `ws://${config.host}:9090` });

ros.on("connection", () => { /* subscribe to topics, start polling node/topic lists */ });
ros.on("error",      (e) => { /* show error, release the handle */ });
ros.on("close",      ()  => { /* tear down subscriptions, release the handle */ });
```

The app auto-connects on load and retries on its own if the connection drops, so a rosbridge restart heals with no user action; a manual **Retry** button covers whatever the automatic retry doesn't.

!!! warning "Lesson learned: stale event handlers from a discarded connection"
    The reconnect logic here originally had two defects, both worth understanding since they're a textbook case of async state bugs, not anything ROS-specific.

    First, the `close`/`error` handlers never cleared the stored connection handle, while `connect()` began with "if a handle already exists, do nothing." After the *first* disconnect, the handle stayed pointing at a dead socket, and every future reconnect attempt silently did nothing — the app just looked permanently stuck.

    Second, calling `.close()` on the old socket fires its `close` event **asynchronously**, potentially *after* a replacement connection is already up. The old handler then ran anyway, tore down the *new* connection's subscriptions, and forced the UI back to "disconnected" even though a working connection existed a moment earlier.

    The fix: release the stored handle inside `close`/`error` themselves, and have every handler check that it still owns the *current* connection before touching shared state. The general lesson travels well beyond this file: **event handlers from a discarded object keep firing — always verify they're still relevant before letting them mutate shared state.**

## Subscribing, publishing, and calling services

**Subscribing to a topic:**

```js
const scan = new ROSLIB.Topic({
  ros,
  name: "/scan",
  messageType: "sensor_msgs/msg/LaserScan",
});
scan.subscribe((msg) => setLidarData(msg));
```

Topics the control interface subscribes to: `/battery_state`, `/odom`, `/scan`, `/map`, `/camera/image_raw/compressed`, `/imu/data_raw`, `/joint_states`.

**Publishing** (velocity commands):

```js
const cmdVel = new ROSLIB.Topic({ ros, name: "/cmd_vel", messageType: "geometry_msgs/msg/Twist" });
cmdVel.publish(new ROSLIB.Message({
  linear:  { x: -linearX, y: 0, z: 0 },
  angular: { x: 0, y: 0, z: -angularZ },
}));
```

Note the negation: this robot's `/cmd_vel` sign convention is reversed from the usual ROS convention. It's negated in exactly **one** place, `ROSProvider`, so every caller — joystick, keyboard, D-pad, distance control — inherits the correction automatically. Normalizing a hardware quirk once, centrally, rather than in every caller, is the right instinct here.

!!! note "The same quirk is corrected twice, independently"
    [`a2bot_service`](a2bot-service.md) fixes this exact sign convention a second time, separately — its `service_server` node defaults `invert_cmd_vel` to `true` and negates every command it publishes. The two corrections don't conflict, because they sit on two different, non-overlapping command paths (this React app publishing `/cmd_vel` directly for manual driving, versus `a2bot_service`'s own services driving the robot for `move_distance`/`turn_angle`). But it does mean the same hardware fact — A2Bot's motors expect the opposite sign from the ROS convention — is encoded independently in two places rather than fixed once at the source (`diff_drive`, see [a2bot_driver](a2bot-driver.md)). Worth knowing if that root cause is ever fixed: both corrections would need removing together, or the robot would end up double-inverted on one of the two paths.

**Calling a service:**

```js
const service = new ROSLIB.Service({
  ros,
  name: "/a2bot/move_distance",
  serviceType: "a2bot_service/srv/MoveDistance",
});
service.callService(new ROSLIB.ServiceRequest({ distance, speed }), onResult, onError);
```

Services the control interface calls, all served by [`a2bot_service`](a2bot-service.md#the-cli-clients)'s `service_server`:

| Service | Type | Purpose |
|---|---|---|
| `/a2bot/move_distance` | `a2bot_service/srv/MoveDistance` | Drive a set distance using odometry feedback |
| `/a2bot/turn_angle` | `a2bot_service/srv/TurnAngle` | Turn a set angle |
| `/a2bot/return_home` | `std_srvs/srv/Trigger` | Reverse the recorded command history |
| `/a2bot/stop` | `std_srvs/srv/Trigger` | Stop motion |
| `/a2bot/relaunch_all` | `std_srvs/srv/Trigger` | Restart all four `a2bot` services |
| `/a2bot/camera_start` | `std_srvs/srv/Trigger` | Start the camera node on demand |
| `/a2bot/camera_stop` | `std_srvs/srv/Trigger` | Stop the camera node |

**Graph introspection** (the live Nodes & Topics panel) uses the `rosapi` services through roslibjs helpers, polled every 2 seconds:

```js
ros.getNodes((nodeList) => setNodes(nodeList));
ros.getTopics((result) => { /* result.topics[], result.types[] */ });
```

Each topic row can also be expanded to echo its live messages — that just opens a temporary subscription for as long as the row stays open, the browser equivalent of `ros2 topic echo`.

**The two HTTP calls that skip rosbridge entirely:** the React app also fetches `http://<host>:8888/api/services/health` (systemd + functional health of the core services, polled every 3 s) and the WiFi endpoints, straight from the FastAPI page. These are deliberately *not* ROS calls: "is the systemd unit running?" is a system question the ROS graph has no way to answer, since a dead service has no ROS presence left to ask.

## Streaming the camera — a performance lesson

The camera subscription isn't a plain subscription — it carries three extra options, each preventing a specific, real failure:

```js
const camera = new ROSLIB.Topic({
  ros,
  name: "/camera/image_raw/compressed",
  messageType: "sensor_msgs/msg/CompressedImage",
  throttle_rate: 100,   // ms — cap at ~10 fps
  queue_length: 1,      // drop stale frames instead of queueing them
  compression: "cbor",  // binary frames, not base64-inside-JSON
});
```

- **`throttle_rate: 100`** — tells rosbridge to forward at most one message per 100 ms.
- **`queue_length: 1`** — the important one. Without it, rosbridge *queues* every frame it can't deliver in time, and the video feed drifts **further and further behind live** the longer it runs. A queue length of 1 drops stale frames instead of hoarding them — latency that keeps growing over time is the classic symptom of an unbounded queue.
- **`compression: "cbor"`** — sends raw binary over the WebSocket instead of base64-encoded text inside JSON, dramatically lighter for image data (base64 costs roughly 33% size overhead, plus JSON parsing on both ends).

Two consequences worth knowing:

1. Always subscribe to `/camera/image_raw/compressed` (JPEG), never raw `/camera/image_raw`. Raw 640×480 video runs roughly **26 MB/s** — more than the Pi or the WiFi link can carry.
2. With CBOR, roslibjs hands back a `Uint8Array` rather than a base64 string, so the app converts it for display **in chunks**. A single `String.fromCharCode.apply(null, hugeArray)` call blows the JavaScript call stack on a full-size frame.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Dashboard shows "Disconnected," nothing loads | rosbridge is not running. `systemctl status a2bot-rosbridge.service`; confirm port 9090 is listening with `ss -ltn \| grep 9090`. It takes ~15 s to accept connections after a restart. |
| Nodes/Topics panel is empty but control works | The `rosapi` node is not running. Publishing and subscribing don't need it; introspection does. |
| Everything "active" in systemd but nodes can't see each other | Mismatched `ROS_DOMAIN_ID` or `RMW_IMPLEMENTATION`. Every node must agree on both. |
| `ros2 node list` times out inside a service | The unit is missing `HOME`/`WorkingDirectory`, so ROS CLI tools can't write to `~/.ros`. |
| Camera feed drifts further behind real time the longer it runs | Missing `queue_length: 1` — rosbridge is queueing frames it can't keep up with. |
| Video feed very slow / low frame rate | Check CPU load with `uptime`. rosbridge is CPU-hungry relaying image data on a Pi; frame rate collapses once the machine is saturated. |
| Camera image looks zoomed in | Aspect ratio, not zoom. libcamera picks the sensor mode from the requested resolution: on the IMX219, a 4:3 request selects the full-sensor 1640×1232 mode, while other ratios can select a **cropped** 1920×1080 mode with a narrower field of view. |

## Quick reference

| Item | Value |
|---|---|
| Control interface (React) | `http://<robot-ip>:7777` |
| Setup / telemetry page (FastAPI) | `http://<robot-ip>:8888` |
| rosbridge WebSocket | `ws://<robot-ip>:9090` |
| `ROS_DOMAIN_ID` | `X`, this robot's own number |
| RMW implementation | `rmw_cyclonedds_cpp` |
| Services | `a2bot-robot`, `a2bot-dashboard`, `a2bot-rosbridge`, `a2bot-webui` |

```bash
# Are the pieces up?
systemctl status a2bot-rosbridge.service
ss -ltn | grep 9090

# Watch what the browser would see
ros2 node list
ros2 topic list
ros2 topic hz /scan

# Restart the bridge
sudo systemctl restart a2bot-rosbridge.service
```

Next: [Part 3 — Setup 1: Raspberry Pi](../part3/setup-1-raspberry-pi.md) or [Setup 2: Laptop](../part3/setup-2-laptop.md).
