# Ubuntu Terminal Basics

## What this is, and why it matters

Both machines in this course — the robot's Raspberry Pi and your own laptop — run Ubuntu, and almost everything you do on either of them happens by typing commands into a terminal rather than clicking through menus. The Pi doesn't even have a screen to click on. This page covers exactly the terminal skills this course actually needs: moving around the filesystem, viewing and editing a file, understanding `sudo`, and reading the flags a command shows you. It is not a general Linux course — if something isn't used anywhere in this project, it isn't here.

## Finding your way around: `pwd`, `ls`, `cd`

A terminal always has a **current directory** — the folder it's "standing in." Three commands are all you need to work with that:

!!! both "🔗 Both"
    ```bash
    pwd
    ```

    Prints the current directory's full path — useful any time you're not sure where you are, especially after following instructions that `cd` around.

    ```bash
    ls
    ```

    Lists what's in the current directory. Add `-l` for a detailed, one-per-line view (permissions, size, modified date), or `-a` to also show hidden files (anything starting with `.`, like `.bashrc`):

    ```bash
    ls -la
    ```

    ```bash
    cd a2bot_ws/src
    ```

    Changes the current directory. `cd ..` moves up one level, and `cd` with no argument (or `cd ~`) jumps straight back to your home directory.

In this course you'll use these constantly just to get around the workspace — for example, `cd ~/a2bot/a2bot_ws` before every `colcon build` in [Setup 1](../part3/setup-1-raspberry-pi.md) and [Setup 2](../part3/setup-2-laptop.md).

## Viewing and editing files: `cat`, `nano`

To print a file's contents straight to the terminal without opening an editor:

!!! both "🔗 Both"
    ```bash
    cat ~/.bashrc
    ```

`cat` is a read-only quick look — this is exactly how the [Troubleshooting Index](../appendices/troubleshooting-index.md) checks things like `cat /etc/netplan/*.yaml` or a running process's environment variables.

To actually edit a file, this course uses **nano** — a small terminal text editor with none of the modal-editing learning curve of something like `vim`:

!!! both "🔗 Both"
    ```bash
    nano ~/.bashrc
    ```

Inside nano, the shortcuts you need are shown at the bottom of the screen with a `^` meaning Ctrl:

| Shortcut | Action |
|---|---|
| `Ctrl+O` | Save (nano calls this "Write Out") — press Enter to confirm the filename |
| `Ctrl+X` | Exit — prompts you to save first if you have unsaved changes |

That's genuinely the whole workflow: open the file, arrow keys to move around and type, `Ctrl+O` then Enter to save, `Ctrl+X` to leave. You'll do exactly this every time a setup step tells you to add a line to `~/.bashrc`.

## `sudo`, and `chmod` where it actually comes up

Most setup and system-configuration commands in this course are prefixed with `sudo` — it runs that one command with administrator privileges, which Ubuntu requires for things like installing packages, editing files outside your own home directory, or configuring the network. You'll type your own account password when prompted (not the robot's SSH password) the first time in a given terminal session.

`chmod` changes a file's permissions, and this course uses it in exactly one place: locking down a sudoers rule so only its intended owner and group can read it (see [Troubleshooting Index — nmcli fails with "not authorized"](../appendices/troubleshooting-index.md#nmcli-fails-with-not-authorized)):

!!! pi "🤖 Pi"
    ```bash
    sudo chmod 0440 /etc/sudoers.d/nmcli
    ```

The `0440` is an octal permission mode: the file's owner can read it, its group can read it, and everyone else — including the file's own owner, for *writing* — gets nothing. Read the digits as three separate audiences (owner, group, everyone else), each digit built from read(4) + write(2) + execute(1) added together; `4` alone means "read-only." You won't need to compute other modes for this course — this is the one real example, and the reasoning generalizes if you ever do need another.

## How command flags work: short vs. long

You'll see two flag styles constantly from here on, and they mean the same kind of thing even though they look different:

- **Short flags**, a single dash plus one letter: `-y`, `-r`, `-a`
- **Long flags**, two dashes plus a word: `--active`, `--symlink-install`, `--from-paths`

Short flags can often be combined behind one dash — `-aG` in a command you'll actually run in [Electronics & Hardware](../part2/electronics-hardware.md) is `-a` and `-G` stuck together:

```bash
sudo usermod -aG dialout,i2c,video,netdev $USER
```

Long flags spell out what they do and are harder to misread, which is why more involved commands lean on them — two you'll type often in [Setup 2](../part3/setup-2-laptop.md):

```bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
```

Note the last example mixes both styles in one command — that's completely normal; a tool's author picks short or long per-flag, usually giving common/simple flags a short form and less-common ones only a long one. There's no way to guess a flag's meaning from its shape alone — `-r` means something different for every tool that defines it — so when a step in this course gives you a flag it hasn't explained, that's deliberate: trust the command as written rather than guessing what to add or remove.

## A find-and-replace tool you'll see once: `sed`

`sed -i` edits a file in place by pattern-matching text, and it shows up in this course exactly once, in the [Troubleshooting Index's rename-artifacts entry](../appendices/troubleshooting-index.md#a-rename-leaves-stale-filesfolders-that-a-text-sweep-cant-fix):

```bash
sed -i 's/A2Bot_description/a2bot_description/g' a2bot_ws/src/a2bot_description/setup.cfg
```

Read as: in this file, replace every (`g`) occurrence of `A2Bot_description` with `a2bot_description`. That's the one pattern worth recognizing on sight — this course doesn't otherwise teach `sed` as a general tool, and doesn't use `awk` at all.

## Where this shows up in the course

Every single step in [Setup 1](../part3/setup-1-raspberry-pi.md) and [Setup 2](../part3/setup-2-laptop.md) assumes you can move around a directory tree, open and edit `~/.bashrc`, and read a `sudo`-prefixed command without it looking unfamiliar. From here on, every `nmcli`, `systemctl`, and `colcon` command you meet will mix short and long flags exactly like the examples above.
