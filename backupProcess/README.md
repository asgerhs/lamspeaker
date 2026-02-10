# Raspberry Pi SD Card Snapshots

## Create a Snapshot (.img)

 1. Shut down the Raspberry Pi by running:
   `sudo shutdown now`

2. Remove the SD card and insert it into a Windows PC using an SD card reader.
   If Windows asks to format the disk, cancel the prompt.

3. Open Win32 Disk Imager as Administrator.

4. Configure the tool:
   - Image File: click the folder icon, and find the correct folder in OneDrive. Give name `pi-{os}-{change/state}-{date in dd-MM-YY format}.img` ex: `pi-bookworm-bluetoothUpdate-10-2-26.img`
   - Device: select the SD card drive letter
   - Read Only Allocated Partitions: leave unchecked

5. Click READ and wait for the process to complete.

The resulting `.img` file is a full snapshot of the system.


## Restore a Snapshot

WARNING: This will overwrite the SD card completely.

1. Insert the SD card into the PC.
2. Open Win32 Disk Imager as Administrator.
3. Select the `.img` file to restore.
4. Select the SD card device.
5. Click WRITE.
6. Safely eject the SD card.
7. Insert it back into the Raspberry Pi and boot.



