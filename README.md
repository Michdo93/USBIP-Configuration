# USBIP-Configuration
How to configure client and server for sharing usb over ip.

## Installation

### Raspberry Pi


### Ubuntu


### Windows



## Configuration

### Client

/etc/systemd/system/usbip-client@.service:

```
[Unit]
Description=USB/IP Client Attach/Detach Service for %i
After=network-online.target
Before=shutdown.target reboot.target halt.target
BindsTo=network.target
After=network.target

[Service]
Type=idle
ExecStartPre=/usr/sbin/modprobe vhci-hcd

ExecStart=/usr/sbin/usbip attach -r 192.168.0.231 -b %i
ExecStop=/usr/bin/sh -c '/usr/sbin/usbip port | grep "<Port in Use>" | awk "{print \$2}" | tr -d ":" | xargs -r /usr/sbin/usbip detach -p'
RemainAfterExit=yes
Restart=on-failure
RestartSec=15s

[Install]
WantedBy=multi-user.target
```

/etc/systemd/system/usbip-watchdog.timer:

```
[Unit]
Description=Timer for USB/IP Client Watchdog

[Timer]
# Der Timer startet 10 Sekunden nach dem Booten
OnBootSec=10s
# Der Timer wird alle 30 Sekunden ausgelöst
OnUnitActiveSec=30s

[Install]
WantedBy=timers.targe
```

/etc/systemd/system/usbip-watchdog.service:

```
[Unit]
Description=USB/IP Client Watchdog Service

[Service]
# Startet das Skript als Root.
ExecStart=/usr/local/bin/usbip-watchdog.sh
```

/usr/local/bin/usbip-watchdog.sh:

```
#!/bin/bash

# Die Ziel-Bus-ID und der Service-Name
BUS_ID="1-1.2"
SERVICE_NAME="usbip-client@$BUS_ID.service"

# Prüfen, ob das Gerät als "In Use" auf Port 00 existiert
# Wenn die Ausgabe leer ist (Gerät nicht gefunden), ist es abgestürzt.
if ! /usr/sbin/usbip port | grep -q "/$BUS_ID"; then
    echo "WATCHDOG: USB/IP Gerät $BUS_ID nicht gefunden. Neustart des Services."
    # Den fehlerhaften Dienst stoppen und neu starten
    /usr/bin/systemctl restart "$SERVICE_NAME"
    exit 1
fi

# Das Gerät ist vorhanden. Alles in Ordnung.
exit 0
```
