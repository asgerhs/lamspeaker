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



## 2. Disable internal Bluetooth & onboard audio

For a festival setup, we want to avoid conflicts and flaky behavior caused by unused hardware.
Since we are using:

- a USB Bluetooth dongle
- an external I2S audio HAT (IQaudIO DigiAMP)

we disable:
- the Raspberry Pi’s internal Bluetooth
- onboard analog audio

This ensures:
- only one Bluetooth controller exists
- predictable audio routing


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



## 3. Adding bluetooth guard service

To ensure we don't get unexpected guests joining the speaker, we want to disallow untrusted devices from joining the speaker when someone has joined. We still want to allow trusted devices. 

To do this, we need to add a custom systemd service. 

See [services/README.md Bluetooth connection guard serice and script](../services/README.md#bluetooth-connection-guard-service-and-script)

To see logs for Bluetooth guard service see [Logs/README.md Viewing Bluetooth guard logs](../Logs/README.md#viewing-bluetooth-guard-logs)


### 2.5 Viewing Bluetooth guard logs

All decisions are logged to journald.

To follow logs live:

    sudo journalctl -f -t bt-guard

This is useful when debugging behavior on-site.


### Notes / gotchas

- After disabling internal Bluetooth, the USB adapter may change from hci1 to hci0.
- If Bluetooth appears dead after reboot, rfkill is usually the cause.
- Trusted devices can reconnect even when pairable is off — this is normal Bluetooth behavior.
