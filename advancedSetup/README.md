# Advanced setup

After doing the initial setup, the speaker should work. Now we just need to make improvements. 


## 1. set sound card by name instead of number. 

While the number setup works, it's a bit fragile as the card number can change. It's not likely, but in a festival setting, you want to be sure that
it doesn't suddenly stop working. Therefore we want to lock it in by name. 

open asound.config again: 

    sudo nano /etc/asound.conf

And replace the card settings with the following: 

    pcm.!default {
    type plug
    slave.pcm "hw:IQaudIODAC"
    }

    ctl.!default {
    type hw
    card "IQaudIODAC"
    }

This assumes that the sound card hasn't been swapped out. To check the name of the sound card run 

    aplay -l

with the current sound card it will say: 

    card 3: IQaudIODAC [IQaudIODAC], device 0: IQaudIO DAC HiFi pcm512x-hifi-0 [IQaudIO DAC HiFi pcm512x-hifi-0]
    Subdevices: 1/1
    Subdevice #0: subdevice #0

then just replace IQaudIODAC with whatever the new HAT is called. 



## 2. Disable internal Bluetooth & HDMI audio, and add Bluetooth auto-recovery

For a festival setup, we want to avoid conflicts and flaky behavior caused by unused hardware.
Since we are using:

- a USB Bluetooth dongle
- an external I2S audio HAT (IQaudIO DigiAMP)

we disable:
- the Raspberry Pi’s internal Bluetooth
- onboard HDMI / analog audio

This ensures:
- only one Bluetooth controller exists
- predictable audio routing
- fewer edge cases after reboot


### 2.1 Disable internal Bluetooth and onboard audio

Open the firmware config:

    sudo nano /boot/firmware/config.txt

Make the following changes:

Disable onboard audio  
Change:

    dtparam=audio=on

to:

    dtparam=audio=off

Disable internal Bluetooth  
Add the following line after `dt-param=audio=off`:

    dtoverlay=disable-bt

Save and reboot:

    sudo reboot


### 2.2 Verify only the USB Bluetooth adapter is active

After reboot, list Bluetooth controllers:

    bluetoothctl list

You should now see only one controller (the USB dongle).

If Bluetooth is blocked after boot:

    rfkill list
    sudo rfkill unblock bluetooth


### 2.3 Bluetooth auto power-on service

After disabling internal Bluetooth, the USB adapter may boot powered off.
To ensure Bluetooth is always available, we add a small systemd service.

Create the service file:

    sudo nano /etc/systemd/system/bt-poweron.service

Contents:

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

Enable the service:

    sudo systemctl daemon-reload
    sudo systemctl enable bt-poweron.service

Verify after reboot:

    bluetoothctl show | grep Powered

Expected:

    Powered: yes


### 2.4 Bluetooth discoverable / pairable guard service

To improve festival UX, we run a guard service that:

- allows pairing when no device is connected
- disables discoverable and pairable when a device is connected
- allows trusted devices to reconnect
- avoids aggressive disconnects to keep audio stable

Service status:

    systemctl status bt-discoverable-guard.service

### 2.4 Bluetooth discoverable / pairable guard service (implementation)

To control discoverability in a festival-friendly way, we run a small background service that continuously:

- checks if any Bluetooth device is currently connected
- turns discoverable + pairable **off** when a device is connected
- turns them **on** again when no devices are connected

This prevents random guests from pairing while someone is already using the speaker,
but still allows fast take-over once playback stops or the device disconnects.


#### 2.4.1 Create the guard script

TODO: this is outdated/incorrect. I think?

Create the script:

    sudo nano /usr/local/bin/bt-discoverable-guard.sh

Contents:

    #!/usr/bin/env bash
    set -euo pipefail

    INTERVAL=3

    log() {
      logger -t bt-guard "$*"
    }

    any_connected() {
      mapfile -t devs < <(
        busctl --system --no-pager --no-legend tree org.bluez 2>/dev/null \
          | grep -oE '/org/bluez/hci[0-9]+/dev_[0-9A-F_]+' \
          | sort -u
      )

      if [ "${#devs[@]}" -eq 0 ]; then
        return 1
      fi

      local connected_count=0
      for dev in "${devs[@]}"; do
        out="$(busctl --system --no-pager --no-legend get-property \
          org.bluez "$dev" org.bluez.Device1 Connected 2>/dev/null || true)"

        if echo "$out" | grep -q 'true$'; then
          connected_count=$((connected_count + 1))
        fi
      done

      [ "$connected_count" -gt 0 ]
    }

    while true; do
      if any_connected; then
        bluetoothctl discoverable off >/dev/null 2>&1 || true
        bluetoothctl pairable off >/dev/null 2>&1 || true
      else
        bluetoothctl discoverable on >/dev/null 2>&1 || true
        bluetoothctl pairable on >/dev/null 2>&1 || true
      fi

      sleep "$INTERVAL"
    done

Make it executable:

    sudo chmod +x /usr/local/bin/bt-discoverable-guard.sh


#### 2.4.2 Create the systemd service

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


#### 2.4.3 Verify behavior

Check service status:

    systemctl status bt-discoverable-guard.service

Check current Bluetooth state:

    bluetoothctl show

Expected behavior:

- No device connected → Discoverable: yes, Pairable: yes
- Device connected → Discoverable: no, Pairable: no



### 2.5 Viewing Bluetooth guard logs

All decisions are logged to journald.

To follow logs live:

    sudo journalctl -f -t bt-guard

This is useful when debugging behavior on-site.


### Notes / gotchas

- After disabling internal Bluetooth, the USB adapter may change from hci1 to hci0.
- If Bluetooth appears dead after reboot, rfkill is usually the cause.
- Trusted devices can reconnect even when pairable is off — this is normal Bluetooth behavior.
