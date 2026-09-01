# Discovering and Controlling A2Bot

This is the final piece of the workshop: once the robot is fully built and
every service is running, how does a student actually *find*, *connect
to*, and *drive* it — with no one standing over their shoulder to help?

## The one convention that ties everything together

Every robot in the classroom is identified by a single number, **X**. That
one number determines *everything* below it — the hostname, the SSH login,
the hotspot name and address, the direct-Ethernet IP, and its ROS2 domain.
Keeping all of these tied to the same number is deliberate: a mismatch
between them (robot 5's hotspot but robot 3's domain ID) is exactly the
kind of confusion that costs real time mid-session. This table is the
canonical robot specification — every other page's per-robot values are
this same pattern, substituting in one robot's actual number.

| Identifier | Pattern | Example (robot 5) |
|---|---|---|
| Hostname | `a2botX-host` | `a2bot5-host` |
| SSH login | `a2botX@a2botX-host` | `a2bot5@a2bot5-host` |
| Hotspot SSID | `a2botX-setup` | `a2bot5-setup` |
| Hotspot address (dashboard) | `10.42.0.X:8888` | `10.42.0.5:8888` |
| Direct-Ethernet IP (`eth0`) | `10.0.0.10X` | `10.0.0.105` |
| `ROS_DOMAIN_ID` | `X` | `5` |

!!! note "Valid domain range"
    `ROS_DOMAIN_ID` must stay between 0 and 101 (values above that collide
    with the OS's ephemeral port range). With more than ~100 robots this
    scheme needs a second dimension — unlikely for a single classroom, but
    worth knowing the ceiling exists.

New to IP addresses, subnets, or `.local` hostnames? See
[IP Addresses & Your Network](../toolkit/ip-addresses-and-your-network.md)
for the concepts this table relies on.

---

## The discovery sequence

This is what a student actually does, start to finish, the first time they
approach a finished robot.

### 1. Find the robot's hotspot

On first boot with no known WiFi, the robot broadcasts its own network:

```
a2botX-setup
```

Connect a laptop or phone to it (password provided separately by the
instructor, or printed on the robot itself).

!!! note "Hotspot not showing up?"
    If `a2botX-setup` doesn't appear in the WiFi list, the robot is
    already connected to a known network from a previous session — so it
    has no reason to broadcast its own hotspot. Press and **hold** the
    physical **WiFi reset** button on the robot until its LED blinks
    (around **5 seconds**) to clear all saved/known networks and force it
    back into hotspot mode.

### 2. Open the dashboard

Two ways in, either works:

- **Scan the QR code** on the robot's chassis — it links straight to the
  dashboard.
- **Type the address directly in the browser:**
  ```
  http://10.42.0.X:8888
  ```
  substituting the robot's actual number for `X`.

This page is the **WiFi setup page** — status of the robot's current
connection, and a form to join a real network.

!!! danger "Disable every other network connection first"
    The dashboard is only reachable while the robot's hotspot is your
    device's **only** active connection. Ethernet, mobile data, or any
    other WiFi network active at the same time will stop the dashboard
    from loading. Turn off mobile data, unplug Ethernet, and make sure
    `a2botX-setup` is the sole connection before opening the dashboard
    address.

### 3. Connect the robot to a personal hotspot

Turn on your phone's personal hotspot, then use the dashboard's **Connect
to WiFi** form to join the robot to it. This gives the robot real internet
access, and puts the robot and your laptop on the *same* network — the
prerequisite for everything after this step.

!!! note "Can't find your personal hotspot in the list?"
    Press the **rescan** button on the dashboard's WiFi form. This makes
    the robot briefly drop its own `a2botX-setup` hotspot to scan for
    nearby networks — so your device will lose its connection to the
    robot while that happens. Reconnect to `a2botX-setup` afterward and
    reopen the dashboard to see the refreshed network list.

!!! warning "You will likely lose the page here — that's expected"
    The instant the robot switches networks, it drops the `a2botX-setup`
    hotspot your browser was using. The connect request appearing to
    "fail" with a network error is the **normal, expected shape of
    success** — the robot switched networks out from under the very
    request that asked it to.

### 4. Reconnect, and get SSH access

Join your own laptop to the same personal hotspot the robot just joined.
Then reopen the dashboard (it will have a new address on this network) —
its post-connect panel shows the exact SSH command and password to use,
so you don't need to remember or look them up:

```bash
ssh a2botX@a2botX-host.local
```

New to SSH itself? See [SSH & Remote Access](../toolkit/ssh-and-remote-access.md)
for what's actually happening here.

### 5. Set up your own ROS2 environment to match this robot

Before your laptop can see the robot's topics and nodes, your shell needs
two things set correctly — **matching the robot's number X**:

!!! note "Cyclone DDS must already be installed"
    This step assumes `rmw_cyclonedds_cpp` is already installed on your
    laptop — see [Setup 2 — Laptop, step 2](../part3/setup-2-laptop.md#2-install-and-configure-cyclone-dds)
    for why it's needed and how to install it. If you haven't done that
    yet, do it before continuing here.

```bash
export ROS_DOMAIN_ID=X
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Add both to `~/.bashrc` so every new terminal has them automatically:

```bash
echo "export ROS_DOMAIN_ID=X" >> ~/.bashrc
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
source ~/.bashrc
```

This `.bashrc` pattern comes up throughout the rest of the course — see
[~/.bashrc](../toolkit/bashrc.md) for why it works this way.

**Verify it worked:**
```bash
ros2 topic list
```
You should see the robot's real topics (`/odom`, `/scan`, `/joint_states`,
and others) — not an empty list. (For a quick reference on commands like
this one, see the [ROS 2 CLI Reference](../toolkit/ros2-cli-reference.md).)

---

## Networking & Service Commands

Once the robot is running, you'll be checking its network status and its
services constantly. Rather than a command table here, see the full
standalone references — useful well beyond this one page:
[nmcli & Network Connections](../toolkit/nmcli-and-network-connections.md)
and [systemd & Services](../toolkit/systemd-and-services.md).

The robot's core software runs as five background services, always
running, starting automatically at boot:

```
a2bot-robot        the driver, description, and lidar
a2bot-dashboard     the web dashboard and WiFi setup page
a2bot-rosbridge      the WebSocket bridge for browser-based control
a2bot-webui           the web control interface
a2bot-gpio             the physical recovery buttons
```

!!! danger "A service being 'active' does not mean it is working"
    `systemctl status` only proves the **process started and stayed
    alive** — it says nothing about whether the robot's actual software
    inside it is functioning. A service can show `active (running)` while
    its ROS nodes have silently failed to initialize, publishing nothing.

    Always cross-check with a **functional** test, not just the process
    check:
    ```bash
    ros2 node list      # are the expected nodes actually present?
    ros2 topic list     # are the expected topics actually publishing?
    ros2 topic hz /odom # is data actually flowing, not just declared?
    ```
    A short diagnostic script exists in the repo (`check_services.py`)
    that runs both checks side by side and reports a service as
    **"running but not functional"** when the two disagree — this exact
    gap has caused real, hard-to-diagnose failures before, so it is worth
    treating as a standard first step, not an afterthought.
