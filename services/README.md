# Services 

## bt-poweron.service

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

### Why this is needed
BlueZ may start with the controller powered off after reboot,
especially when using USB Bluetooth dongles.
This service runs once at boot and safely powers the controller on.

### How to inspect
    systemctl status bt-poweron.service

### How to disable
    sudo systemctl disable bt-poweron.service
