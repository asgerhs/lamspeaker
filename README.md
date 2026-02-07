# Raspberry Pi Bluetooth Speaker Setup (DigiAMP+)

This document describes the exact steps used to set up a Raspberry Pi–based
Bluetooth speaker with a DigiAMP+ (IQaudIO / HiFiBerry-compatible) HAT, using
`rpi-audio-receiver`, with phone-controlled volume and a safe hardware volume
ceiling.

The goal is to establish a **stable baseline** before adding knobs, displays,
power management, or read-only filesystems.


## 1. Install the Operating System

### Requirements
- Raspberry Pi OS (Legacy) Lite
- Debian 12 (Bookworm)
- 32-bit or 64-bit (64-bit used here)

IMPORTANT:
Do NOT use plain Debian.
The OS must identify as **Raspberry Pi OS**, not just `Debian`.

### Verify OS after boot

    lsb_release -a

Expected output:

    Distributor ID: RaspberryPi
    Release:        12
    Codename:       bookworm


## 2. Install rpi-audio-receiver (Bluetooth only)

Run from the home directory:

    cd ~
    wget https://raw.githubusercontent.com/nicokaiser/rpi-audio-receiver/main/install.sh
    bash install.sh

During installation:
- Bluetooth: YES
- AirPlay: NO
- Spotify: NO
- Set hostname / pretty name (e.g. `lamspeaker`)

Do NOT run the script with `sudo`.

## 3. Set a Safe Hardware Volume (Ceiling)

Open the ALSA mixer:

    alsamixer

Inside `alsamixer`:
- Press F6
- Select `IQaudIODAC`
- Adjust `Digital` volume to a safe level (e.g. 30–50%)

Exit with `Esc`.

Persist the mixer state:

    sudo alsactl store

This sets a **hardware gain ceiling** to protect speakers and ears.

## 4. Enable the DigiAMP+ (I2S Audio HAT)

Edit firmware configuration:

    sudo nano /boot/firmware/config.txt

Add the following line (do not remove other lines yet):

    dtoverlay=hifiberry-digiamp

Reboot:

    sudo reboot

### Verify that the HAT is detected

    aplay -l

Expected to see something like:

    card X: IQaudIODAC [IQaudIODAC], device 0: ...

Note the **card number** (used in the next step).


## 5. Set the DigiAMP+ as the Default ALSA Output

Create or edit `/etc/asound.conf`:

    sudo nano /etc/asound.conf

Set defaults to the DigiAMP+ card number  
(example assumes the HAT is card 3):

    defaults.pcm.card 3
    defaults.ctl.card 3

### Verify audio output

    speaker-test -t sine -f 440 -c 2 -l 1

You should hear a test tone through the speaker.





## 6. Enable Bluetooth A2DP Volume Control

Create a systemd override for Bluetooth:

    sudo mkdir -p /etc/systemd/system/bluetooth.service.d
    sudo nano /etc/systemd/system/bluetooth.service.d/override.conf

Add EXACTLY this:

    [Service]
    ExecStart=
    ExecStart=/usr/libexec/bluetooth/bluetoothd --plugin=a2dp

Reload and restart Bluetooth:

    sudo systemctl daemon-reload
    sudo systemctl restart bluetooth


## 7. Persist the Bluetooth Device Name

Edit Bluetooth configuration:

    sudo nano /etc/bluetooth/main.conf

Ensure the following exists:

    [General]
    Name = lamspeaker

    [Policy]
    AutoEnable=true

Restart Bluetooth:

    sudo systemctl restart bluetooth


## 8. Recover Bluetooth After Restarts (Important)

After restarting Bluetooth, the controller may be powered off.

If you cannot connect:

    bluetoothctl power on

If needed:

    bluetoothctl pairable on
    bluetoothctl discoverable on

Verify state:

    bluetoothctl show

Expected:

    Powered: yes
    Pairable: yes
    Name: lamspeaker
    Alias: lamspeaker


## 9. Resulting Setup (Baseline)

At this point you have:
- DigiAMP+ working over I2S
- Bluetooth A2DP audio
- Phone-controlled volume
- Safe hardware volume ceiling
- Persistent Bluetooth name
- Stable reconnection behavior

### Intentionally NOT configured yet
- No softvol / dmix pipeline
- No potentiometer
- No display integration
- No power-button handling
- No read-only filesystem

These are future steps built on top of this baseline.


## Recovery Notes

- The setup is intentionally stateless
- If something breaks badly, reflashing and re-running these steps is fastest
- This baseline should always be reproducible
