# Services 

## Bluetooth Power On Service

### Name: bt-poweron.service

### Purpose
Forces the Bluetooth controller to power on automatically after boot.
Prevents the system from starting with Bluetooth in "Powered: no" state.

### Location
    /etc/systemd/system/bt-poweron.service

### Contents
    [Unit]
    Description=Force Bluetooth controller power on
    After=bluetooth.target
    Wants=bluetooth.target

    [Service]
    Type=oneshot
    ExecStart=/bin/sleep 2
    ExecStart=/usr/bin/bluetoothctl power on

    [Install]
    WantedBy=multi-user.target

### Enable the service:

    sudo systemctl daemon-reload
    sudo systemctl enable bt-poweron.service

### Verify after reboot:

    bluetoothctl show | grep Powered

### Why this is needed
BlueZ may start with the controller powered off after reboot,
especially when using USB Bluetooth dongles.
This service runs once at boot and safely powers the controller on.

### How to inspect
    systemctl status bt-poweron.service

### How to disable
    sudo systemctl disable bt-poweron.service


## Bluetooth connection guard service and script

### Name: bt-discoverable-guard.serivce

### Purpose
Since we allow multiple connections at once, we want to prevent untrusted people to connect. 
Additionally we also hide the speaker, so it's not visible to others while in use who aren't already trusted parties. Although we only allow one source for sound at a time, we don't want to block the speaker because someone is connected but not playing any sound. 

### location 

    /usr/local/bin/bt-discoverable-guard.sh
    /etc/systemd/system/bt-discoverable-guard.service

### 1. Script

the script will check for connected devices, and if any devices are connected, it run a bluetoothctl command to turn pairing and visibility on/off.

Create the script:

    sudo nano /usr/local/bin/bt-discoverable-guard.sh

Contents:

    #!/usr/bin/env bash
    set -euo pipefail

    # How often to re-check connection state (seconds)
    INTERVAL=3

    log() {
    logger -t bt-guard "$*"
    }


    # Return 0 if any device is connected, else 1
    any_connected() {
    mapfile -t devs < <(
        busctl --system --no-pager --no-legend tree org.bluez 2>/dev/null \
        | grep -oE '/org/bluez/hci[0-9]+/dev_[0-9A-F_]+' \
        | sort -u
    )

    if [ "${#devs[@]}" -eq 0 ]; then
        log "any_connected: no dev_ paths found"
        return 1
    fi

    log "any_connected: checking ${#devs[@]} devices"

    local connected_count=0
    for dev in "${devs[@]}"; do
        # example output: b true / b false
        local out
        out="$(busctl --system --no-pager --no-legend get-property org.bluez "$dev" org.bluez.Device1 Connected 2>/dev/null || true)"

        # If we can't read it, log that too
        if [ -z "$out" ]; then
        log "  $dev Connected=? (no output)"
        continue
        fi

        log "  $dev Connected=$out"

        if echo "$out" | grep -q 'true$'; then
        connected_count=$((connected_count + 1))
        fi
    done

    if [ "$connected_count" -gt 0 ]; then
        log "any_connected: TRUE (connected_count=$connected_count)"
        return 0
    fi

    log "any_connected: FALSE (connected_count=0)"
    return 1
    }


    while true; do
    if any_connected; then
        log "action: discoverable off (device connected)"
        bluetoothctl discoverable off >/dev/null 2>&1 || true
        bluetoothctl pairable off >/dev/null 2>&1 || true
    else
        log "action: discoverable on (no device connected)"
        bluetoothctl discoverable on >/dev/null 2>&1 || true
        bluetoothctl pairable on >/dev/null 2>&1 || true
    fi

    sleep "$INTERVAL"
    done


Make it executable:

    sudo chmod +x /usr/local/bin/bt-discoverable-guard.sh

### 2. Service

Create the service file:

    sudo nano /etc/systemd/system/bt-discoverable-guard.service

Contents:

    [Unit]
    Description=Bluetooth discoverability guard
    After=bluetooth.target
    Wants=bluetooth.target

    [Service]
    Type=simple
    ExecStart=/usr/local/bin/bt-discoverable-guard.sh
    Restart=always
    RestartSec=2

    [Install]
    WantedBy=multi-user.target

Enable and start the service:

    sudo systemctl daemon-reload
    sudo systemctl enable bt-discoverable-guard.service
    sudo systemctl start bt-discoverable-guard.service


### 3. Verify

Check service status:

    systemctl status bt-discoverable-guard.service

Check current Bluetooth state:

    bluetoothctl show

Expected behavior:

- No device connected → Discoverable: yes, Pairable: yes
- Device connected → Discoverable: no, Pairable: no