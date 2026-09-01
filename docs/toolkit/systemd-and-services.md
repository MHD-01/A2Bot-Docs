# systemd & Services

## What this is, and why it matters

systemd is Ubuntu's service manager — it starts, stops, and supervises long-running background processes ("services," defined by **unit files**) automatically, including right at boot with no one logged in. This project runs its own software this way: `a2bot-robot`, `a2bot-dashboard`, `a2bot-rosbridge`, `a2bot-webui`, and `a2bot-gpio` are all systemd services on the Pi, specifically so a student never has to SSH in and manually start five different programs by hand every time the robot powers on.

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

## What "active" actually proves — and what it doesn't

`systemctl status` reporting `active (running)` only proves the process **started and has stayed alive**. It says nothing about whether the software inside it is actually doing anything useful — a process can be alive, using real CPU, and still be silently non-functional underneath. Two entries in the [Troubleshooting Index](../appendices/troubleshooting-index.md) are direct consequences of that gap, and this page's job is just to make the mechanism behind both make sense, not to repeat their fixes:

- [A systemd service comes up "active" on the wrong ROS_DOMAIN_ID / RMW_IMPLEMENTATION](../appendices/troubleshooting-index.md#a-systemd-service-comes-up-active-on-the-wrong-ros_domain_id-rmw_implementation) — a real example of "active" hiding a genuine, silent misconfiguration.
- [Prefer an active readiness check over a fixed sleep for service startup ordering](../appendices/troubleshooting-index.md#prefer-an-active-readiness-check-over-a-fixed-sleep-for-service-startup-ordering) — why one service depending on another needs more than `Requires=`/`After=` ordering alone, since neither proves the dependency is truly *ready*, only that it *started*.

## Where this shows up in the course

[Discovery & Dashboard](../part3/discovery-and-dashboard.md) and [Setup 1 — Raspberry Pi](../part3/setup-1-raspberry-pi.md) both assume you can check whether the robot's own services are up and read their logs when something isn't behaving as expected — this page is the reference for exactly that, any time a setup step or troubleshooting entry just says "check the service."
