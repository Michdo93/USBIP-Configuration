# USBIP-Configuration
How to configure client and server for sharing usb over ip.

## Installation

### Raspberry Pi

```
sudo apt update
sudo apt install usbip
```

### Ubuntu

```
sudo apt install linux-tools-generic
# or
sudo apt install linux-tools-$(uname -r)
```

### Windows

You have to use `Windows Subsystem on Linux 2 (WSL 2)`. After that follow the Ubuntu installation guide.

## Configuration

> **_NOTE:_**  On Ubuntu `/usr/sbin/...` changes to `/usr/bin/...`

### Server

Loading kernel modules:

```
# temporarly
sudo modprobe usbip_host
# permanently
echo "usbip_host" | sudo tee -a /etc/modules
```

/etc/systemd/system/usbipd.service:

```
[Unit]
Description=USB/IP Daemon
# Lädt das Host-Modul vor dem Start des Daemons
# und wartet auf das Netzwerk
Requires=network.target
After=network.target

[Service]
# ExecStartPre wird vor ExecStart ausgeführt
ExecStartPre=/usr/sbin/modprobe usbip_host
# Startet den Daemon im Hintergrund (-D)
ExecStart=/usr/sbin/usbipd
# Stellt sicher, dass der Prozess mit dem Daemon-Kommando ausgeführt wird
Type=simple
# Wichtig: Startet den Dienst neu, wenn er fehlschlägt
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

/etc/systemd/system/usbip-server-bind@.service

```
[Unit]
Description=USB/IP Server Bind for Bus ID %i
After=local-fs.target sysinit.target
RequiresMountsFor=/sys

After=network.target usbipd.service
Requires=usbipd.service

[Service]
Type=simple
ExecStartPre=/usr/sbin/modprobe usbip_host
ExecStart=/usr/sbin/usbip bind -b %i
ExecStop=/usr/sbin/usbip unbind -b %i
RemainAfterExit=yes
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

At first you have to start and enable the `usbip.service` for running the USB/IP Daemon:

```
sudo systemctl start usbip.service
sudo systemctl enable usbip.service
```

Then you have to release an USB port. As example the port `1-1.2`:

```
sudo systemctl start usbip-server-bind@1-1.2.service
sudo systemctl enable usbip-server-bind@1-1.2.service
```

### Client

Loading kernel modules:

```
# temporarly
sudo modprobe vhci-hcd
# permanently
echo "vhci-hcd" | sudo tee -a /etc/modules
```

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
WantedBy=timers.target
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

At first you have to connect to the USB/IP port you have released on your server:

```
sudo systemctl start usbip-client@1-1.2.service
sudo systemctl enable usbip-client@1-1.2.service
```

After that you have to start the watchdog. The watchdog will restart your service if the server is down and restarted a few seconds later.

```
sudo systemctl enable usbip-watchdog.timer
sudo reboot
```

## Conclusion

You can restart both the client and the server. The USB connection via USB/IP should be reestablished each time.
