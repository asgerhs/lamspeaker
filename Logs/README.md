### Viewing Bluetooth guard logs

All decisions are logged to journald.

To follow logs live:

    sudo journalctl -f -t bt-guard

This is useful when debugging behavior on-site.