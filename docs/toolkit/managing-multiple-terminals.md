# Managing Multiple Terminals

## What this is, and why it matters

Driving A2Bot rarely fits in one terminal. A typical session has one running the robot's launch file, another running teleop to actually drive it, a third with RViz's output or a `ros2 topic echo`, and maybe a fourth just watching logs — all at once, all needing to stay visible while you work in the others. **Terminator** is a graphical terminal application built for exactly this: one window, split into as many independent terminal panes as you need, arranged with the mouse or a keyboard shortcut — no command syntax to memorize.

## Installing it

!!! laptop "💻 Laptop"
    ```bash
    sudo apt install terminator
    ```

## Using it

Launch it from your applications menu, or by typing `terminator` in an existing terminal. A fresh window starts as a single pane — split it as you go:

| Shortcut | Action |
|---|---|
| `Ctrl+Shift+E` | Split the current pane vertically (side by side) |
| `Ctrl+Shift+O` | Split the current pane horizontally (top/bottom) |
| `Ctrl+Shift+Right/Left/Up/Down` | Move focus to the pane in that direction |
| `Ctrl+Shift+W` | Close the current pane |
| `Ctrl+Shift+T` | Open a new tab — a separate group of panes, switchable along the top of the window |

Every one of these is also available by **right-clicking** inside any pane, if you'd rather use a menu than remember a shortcut. Panes can be resized by dragging the thin divider between them with the mouse, just like resizing any other split view.

Each pane is a completely ordinary terminal — anything you'd normally type (`ssh`, `ros2 launch`, `nano`) works exactly the same inside one. The only difference from separate terminal windows is that they're all visible together, arranged how you want, instead of buried in a taskbar.

## Where this shows up in the course

[Bringup & Driving](../part3/bringup-and-driving.md) and [SLAM & Navigation](../part3/slam-and-navigation.md) both assume several terminals running at once — a launch file, teleop, and a viewer, at minimum. Terminator isn't mandated: separate terminal windows do the same job for a short session — it just keeps things organized once the number of terminals stops fitting comfortably on screen.
