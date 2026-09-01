# VS Code Remote Development

## What this is, and why it matters

Plain SSH (see [SSH & Remote Access](ssh-and-remote-access.md)) gives you a terminal on the Pi, but a terminal alone means editing files with `nano` and no syntax highlighting, no file tree, none of the editor conveniences you're used to. VS Code's **Remote-SSH** extension bridges that gap: it connects to the Pi over the exact same SSH mechanism, then lets you browse the Pi's filesystem, edit its files, and run a terminal on it — all from a normal VS Code window on your laptop, as if the Pi's files were local.

This page assumes VS Code and the Remote-SSH extension are **already installed** on your laptop — that one-time install isn't covered here, only what to do once it's in place. It also assumes you already understand the `ssh <user>@<host>` command itself; see [SSH & Remote Access](ssh-and-remote-access.md) for the underlying mechanics (the host-authenticity prompt, password auth, and the `.local` hostname fallback) rather than having them re-explained on this page.

## Connecting to the robot

With the extension installed, open VS Code's command palette (`Ctrl+Shift+P` / `Cmd+Shift+P`) and run:

```
Remote-SSH: Connect to Host...
```

When prompted for a host, type it exactly as you would to `ssh` directly — the robot's username and `.local` hostname built from its number:

```
a2botX@a2botX-host.local
```

The first time you connect to a given robot, VS Code will show you the same "authenticity of host" confirmation `ssh` itself shows on the command line — accept it the same way, and then enter the robot's password when asked. VS Code opens a new window once the connection succeeds, with a green indicator in the bottom-left corner showing which host you're connected to — that indicator is worth glancing at before running anything, so you always know whether a given window is touching the Pi or your own laptop.

## Opening a remote folder

A fresh connection opens with no folder loaded. Use **File → Open Folder** and navigate to the workspace on the Pi — `~/a2bot/a2bot_ws` is the one you'll want for this course. Once open, the Explorer sidebar on the left shows the Pi's actual filesystem, not your laptop's — you're browsing and editing files that physically live on the robot.

## Editing files directly on the Pi

Any file you open through that Explorer — a launch file, a config YAML, `~/.bashrc` — opens in a normal VS Code editor tab. Saving it (`Ctrl+S`) writes the change straight back to the Pi over the SSH connection; there's no separate upload or sync step. This is the same end result as editing with `nano` over a plain SSH session, just with a GUI editor instead of a terminal one — pick whichever fits what you're doing at the time.

## The integrated terminal

**Terminal → New Terminal** opens a shell that runs **on the Pi**, not your laptop — the same as if you'd `ssh`'d in directly, but living inside the editor window next to the files you're working on. This is genuinely useful once you're editing something and immediately want to test it: change a launch file in the editor tab, then `ros2 launch` it in the terminal panel right below, without switching windows.

## Where this shows up in the course

Nothing in this course strictly *requires* VS Code's Remote-SSH — everything it does, plain `ssh` plus `nano` can also do (see [Ubuntu Terminal Basics](ubuntu-terminal-basics.md)). It's offered as a more comfortable option for the parts of [Setup 1](../part3/setup-1-raspberry-pi.md) and [Setup 2](../part3/setup-2-laptop.md) that involve editing files on the Pi or watching its output while changing something, not as a separate required step.
