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