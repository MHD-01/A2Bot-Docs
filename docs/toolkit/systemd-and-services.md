# systemd & Services

## What this is, and why it matters

systemd is Ubuntu's service manager — it starts, stops, and supervises long-running background processes ("services," defined by **unit files**) automatically, including right at boot with no one logged in. This project runs its own software this way: `a2bot-robot`, `a2bot-dashboard`, `a2bot-rosbridge`, `a2bot-webui`, `ros2-ready`, and `a2bot-gpio` are all systemd services on the Pi, specifically so a student never has to SSH in and manually start six different programs by hand every time the robot powers on — see [Setup 1 — Raspberry Pi, step 15](../part3/setup-1-raspberry-pi.md#15-set-up-the-systemd-services) for what each one does.

## The core commands

!!! pi "🤖 Pi"
    ```bash
    systemctl status a2bot-robot
    ```

    Is it running, and since when — the everyday health check.

    ```bash
    sudo systemctl start a2bot-robot
    sudo systemctl stop a2bot-robot
    sudo systemctl restart a2bot-robot
    ```

    Start, stop, or restart (stop then start) a service right now, for this boot only.

    ```bash
    sudo systemctl enable a2bot-robot
    sudo systemctl disable a2bot-robot
    ```

    Make a service start automatically at every future boot, or stop it from doing so — independent of whether it's running *right now*.

    ```bash
    sudo systemctl enable --now a2bot-robot
    ```

    Both at once: start it now, and make it start at every boot from here on.

Reading its output, rather than just whether it's running:

!!! pi "🤖 Pi"
    ```bash
    journalctl -u a2bot-robot -f
    ```

    Follow a service's log live, as new lines are written — press `Ctrl+C` to stop watching.

    ```bash
    journalctl -u a2bot-robot -n 50 --no-pager
    ```

    Print just its last 50 log lines and exit, rather than opening an interactive pager.

## Anatomy of a unit file, and writing one from scratch

Most of the time you're only ever pointing the commands above at a service someone else already wrote. Occasionally you need to write one yourself, or edit an existing `.service` file that ships with this project's software — this section is the general reference for that.

A unit file is a plain-text config file, one per service, split into three sections:

```ini
[Unit]
Description=Short human-readable description of what this does
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/some-command --with-flags
Environment=SOME_VAR=value
WorkingDirectory=/home/user/some/directory
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- **`[Unit]`** — metadata and ordering, not the program itself. `Description=` is what shows up in `systemctl status`. `After=`/`Wants=`/`Requires=` control *when* this service starts relative to others — but only when, not whether the thing it's ordered after is actually ready; see [What "active" actually proves](#what-active-actually-proves-and-what-it-doesnt) below for why that distinction matters.
- **`[Service]`** — what actually runs. `ExecStart=` is the command systemd launches. `Type=simple` (the default, and the only type this project uses) means systemd considers the service "started" the instant that process launches, with no readiness signal expected back. `Environment=` sets one environment variable per line — this is the only reliable way to hand a systemd-managed process a variable that would otherwise come from `~/.bashrc`, since a unit does not source it (see [~/.bashrc — Where this bites](bashrc.md#where-this-bites-in-this-project-specifically)). `WorkingDirectory=` sets the directory the process runs from, equivalent to `cd`-ing there first. `Restart=on-failure` restarts the process automatically if it crashes, but not if you stop it deliberately with `systemctl stop`.
- **`[Install]`** — what `enable` actually wires up. `WantedBy=multi-user.target` means "start this at normal multi-user boot" — the standard choice for a background service with no GUI dependency, which covers every service this project runs.

### Where it goes

!!! pi "🤖 Pi"
    ```bash
    sudo nano /etc/systemd/system/my-service.service
    ```

    `/etc/systemd/system/` is where locally-written or locally-installed unit files live — distinct from `/lib/systemd/system/`, which holds units that came from an apt package. Anything you write or copy in for this project belongs in `/etc/systemd/system/`.

### Bringing it to life {#bringing-it-to-life}

Every one of these steps is necessary, in this order, whether you just wrote a brand-new unit or only edited an existing one:

!!! pi "🤖 Pi"
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now my-service
    systemctl status my-service
    ```

    - `daemon-reload` tells systemd to re-read unit files from disk — without it, systemd keeps using whatever it last loaded into memory, so an edit you just saved has **no effect** until this runs, with no error to explain why.
    - `enable --now` both starts it immediately and makes it start at every future boot — see [The core commands](#the-core-commands) above if you only want one half of that.
    - `status` confirms it's actually running — remember that "active" alone doesn't prove the software inside is *working*, only that it started; see below.

Edited a unit file and nothing seems to have changed? A missing `daemon-reload` is the first thing to check — it's a common, silent way for a change to appear to do nothing.

## What "active" actually proves — and what it doesn't

`systemctl status` reporting `active (running)` only proves the process **started and has stayed alive**. It says nothing about whether the software inside it is actually doing anything useful — a process can be alive, using real CPU, and still be silently non-functional underneath. Two entries in the [Troubleshooting Index](../appendices/troubleshooting-index.md) are direct consequences of that gap, and this page's job is just to make the mechanism behind both make sense, not to repeat their fixes:

- [A systemd service comes up "active" on the wrong ROS_DOMAIN_ID / RMW_IMPLEMENTATION](../appendices/troubleshooting-index.md#a-systemd-service-comes-up-active-on-the-wrong-ros_domain_id-rmw_implementation) — a real example of "active" hiding a genuine, silent misconfiguration.
- [Prefer an active readiness check over a fixed sleep for service startup ordering](../appendices/troubleshooting-index.md#prefer-an-active-readiness-check-over-a-fixed-sleep-for-service-startup-ordering) — why one service depending on another needs more than `Requires=`/`After=` ordering alone, since neither proves the dependency is truly *ready*, only that it *started*.

## Where this shows up in the course

[Discovery & Dashboard](../part3/discovery-and-dashboard.md) and [Setup 1 — Raspberry Pi](../part3/setup-1-raspberry-pi.md) both assume you can check whether the robot's own services are up and read their logs when something isn't behaving as expected — this page is the reference for exactly that, any time a setup step or troubleshooting entry just says "check the service."
