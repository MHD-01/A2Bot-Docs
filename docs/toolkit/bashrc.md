# `~/.bashrc`

## What this is, and why it matters

`~/.bashrc` is a script bash runs automatically every time you open a new **interactive** terminal — before you type anything, before your first prompt even appears. Anything it sets stays set for every command you run in that terminal, which makes it the natural place to put setup that would otherwise be tedious to repeat by hand.

## Why this course needs it

Every terminal that runs a `ros2` command needs ROS 2 "sourced" first:

!!! both "🔗 Both"
    ```bash
    source /opt/ros/humble/setup.bash
    ```

Without it, `ros2` isn't a recognized command at all. Typing that by hand in every single new terminal — and this course routinely has several open at once (see [Managing Multiple Terminals](managing-multiple-terminals.md)) — gets old within your first session. [Setup 1](../part3/setup-1-raspberry-pi.md) and [Setup 2](../part3/setup-2-laptop.md) solve this the same way, over and over: append the `source` line (plus a couple of `export` lines, for things like `ROS_DOMAIN_ID`) to `~/.bashrc` once, and every terminal you open afterward has it automatically:

!!! both "🔗 Both"
    ```bash
    echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ```

The second line reloads `~/.bashrc` in your *current* terminal, so you don't have to close and reopen it to see the effect — new terminals would have picked it up automatically either way.

## Interactive vs. non-interactive: the guard at the top of `.bashrc`

Open `~/.bashrc` (with `nano`, see [Ubuntu Terminal Basics](ubuntu-terminal-basics.md)) and near the very top you'll find something like this:

```bash
case $- in
    *i*) ;;
      *) return;;
esac
```

This checks whether the current shell is **interactive** — roughly, "is a human typing into this, with a prompt" — and if it isn't, the script exits immediately via `return`, before reaching any of the `export`/`source` lines you or a setup step added further down. An interactive terminal you open yourself always passes this check without you noticing it's there. The distinction only starts to matter once *something else* starts a shell on your behalf.

A **login shell** is a related but different concept — it's the kind of shell you get logging into a machine fresh (or via `ssh`, or `bash -l`) — and it is not the same thing as an interactive one. A login shell started non-interactively (for example, a systemd unit running `ExecStart=/bin/bash -l -c '...'`) is a real, common case that fails this exact `$-` check and hits `return` before your exports ever run — silently, with no error.

## Where this bites, in this project specifically

This isn't a hypothetical — it's the root cause behind two entries in the [Troubleshooting Index](../appendices/troubleshooting-index.md), for two different variables:

- [A systemd service comes up "active" on the wrong ROS_DOMAIN_ID / RMW_IMPLEMENTATION](../appendices/troubleshooting-index.md#a-systemd-service-comes-up-active-on-the-wrong-ros_domain_id-rmw_implementation) — a service started this way can come up fully alive while silently missing the `ROS_DOMAIN_ID`/`RMW_IMPLEMENTATION` this project's `.bashrc` sets, because it never reached that line.
- [Not covered here](../appendices/troubleshooting-index.md#not-covered-here) — the equivalent gotcha for `PATH`, in a project that runs a Node.js process under systemd (`nvm`'s own `.bashrc` fix has the same non-interactive blind spot).

Both entries cover their own specific diagnosis and fix in full — this page's job is only to make the underlying mechanism make sense once you get there, not to repeat either fix here.

## Where this shows up in the course

Every `export ROS_DOMAIN_ID=...` and `export RMW_IMPLEMENTATION=...` step in [Setup 1](../part3/setup-1-raspberry-pi.md), [Setup 2](../part3/setup-2-laptop.md), and [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) is really just "append a line to `~/.bashrc`," relying on exactly the mechanism this page describes. If a terminal you opened yourself ever seems to be missing a variable you're sure you set, `cat ~/.bashrc` (see [Ubuntu Terminal Basics](ubuntu-terminal-basics.md)) to confirm the line is actually there before assuming something more exotic is wrong.
