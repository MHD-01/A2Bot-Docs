# nmcli & Network Connections

## What this is, and why it matters

`nmcli` is the command-line tool for NetworkManager, Ubuntu's network configuration service — checking what's connected, scanning for networks, joining one, or forgetting one, all without a GUI. That matters specifically because the Pi has no GUI at all: `nmcli` (wrapped by this project's own dashboard) is genuinely the only way its WiFi gets configured.

## Connection profiles vs. SSIDs

NetworkManager doesn't think in terms of "the network you're on" as a single fact — it stores a **connection profile**: a saved configuration with its own name, created the first time you join a network. That profile's name is *usually* the same as the network's visible SSID, but not always — reconnecting or scripting against "the profile" rather than "the SSID" is where this distinction actually matters, since a command referring to a profile name that doesn't match the real one fails with "connection not found" even though the network itself is right there.

## Checking status

!!! both "🔗 Both"
    ```bash
    nmcli device status
    ```

    Every network interface and its current state — which one is WiFi, which is Ethernet, and whether each is connected, disconnected, or unavailable.

    ```bash
    nmcli connection show --active
    ```

    Just the connection(s) actually active right now, rather than every profile ever saved.

## Scanning for networks

!!! both "🔗 Both"
    ```bash
    nmcli device wifi list --rescan yes
    ```

`--rescan yes` forces a fresh over-the-air scan rather than returning NetworkManager's possibly-stale cached list.

## Connecting

!!! both "🔗 Both"
    ```bash
    nmcli connection up "<profile-name>"
    ```

    Reconnects to a network you've joined before, by its saved profile name.

    ```bash
    sudo nmcli device wifi connect "<SSID>" password "<password>"
    ```

    Joins a network for the first time, creating a new profile in the process. `sudo` is required here — joining or hosting a network is a privileged NetworkManager operation; see [Troubleshooting Index — nmcli fails with "not authorized"](../appendices/troubleshooting-index.md#nmcli-fails-with-not-authorized) if this fails outright rather than just asking for a password.

## Forgetting a network

!!! both "🔗 Both"
    ```bash
    sudo nmcli connection delete "<profile-name>"
    ```

    Removes a saved profile entirely — the device won't auto-reconnect to it again until you join it fresh.

## The one hard constraint: one radio, one job

A single WiFi radio can be a connected client, *or* an access point broadcasting its own network — never both at the same instant. Scanning while an access point is up requires taking that access point down first, which is exactly the situation this project's own hotspot hits every time someone uses its "rescan" button — see [Troubleshooting Index — The WiFi radio can't scan while the setup hotspot is up](../appendices/troubleshooting-index.md#the-wifi-radio-cant-scan-while-the-setup-hotspot-is-up) for how that's actually handled.

## Where this shows up in the course

The robot's own dashboard uses exactly these operations under the hood for its WiFi setup page — see [Discovering and Controlling A2Bot](../part0/discovering-and-controlling.md) for the guided version, and [IP Addresses & Your Network — Direct Ethernet link: a fixed private network](ip-addresses-and-your-network.md#direct-ethernet-link-a-fixed-private-network) for `nmcli`'s role in configuring a wired connection instead of WiFi.
