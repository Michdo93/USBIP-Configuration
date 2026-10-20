# USB/IP Configuration
How to configure client and server for sharing usb over ip.

## Installation

### Raspberry Pi

```bash
sudo apt update
sudo apt install usbip

```

### Ubuntu

```bash
sudo apt install linux-tools-generic
# or
sudo apt install linux-tools-$(uname -r)

```

### Windows

You have to use `Windows Subsystem on Linux 2 (WSL 2)`. After that follow the Ubuntu installation guide.

## Configuration

> ***NOTE:***  On Ubuntu `/usr/sbin/...` changes to `/usr/bin/...` (or check with `which usbip`).

### Useful Diagnostic Commands

To check exportable, local, or remote devices, you can use the following commands:

```bash
# List local USB devices (to find your Bus-IDs)
usbip list --local
# or with full path if needed:
sudo /usr/sbin/usbip list -l

# Check which devices are currently exported/published locally on the server
sudo /usr/sbin/usbip list --remote 127.0.0.1

# Check which devices are provided by a specific remote server
usbip list --remote 192.168.0.231

```

### Server

Loading kernel modules:

```bash
# temporarly
sudo modprobe usbip_host
# permanently
echo "usbip_host" | sudo tee -a /etc/modules

```

`/etc/systemd/system/usbipd.service`:

```ini
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

`/etc/systemd/system/usbip-server-bind@.service`:

```ini
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

At first you have to start and enable the `usbipd.service` for running the USB/IP Daemon:

```bash
sudo systemctl start usbipd.service
sudo systemctl enable usbipd.service

```

Then you have to release a USB port. As an example the port `1-1.2`:

```bash
sudo systemctl start usbip-server-bind@1-1.2.service
sudo systemctl enable usbip-server-bind@1-1.2.service

```

### Client

Loading kernel modules:

```bash
# temporarly
sudo modprobe vhci-hcd
# permanently
echo "vhci-hcd" | sudo tee -a /etc/modules

```

#### Connecting multiple servers to the client (Recommended Template Setup)

`/etc/systemd/system/usbip-client@.service`:

```ini
[Unit]
Description=USB/IP Client Attach/Detach Service for %i
After=network-online.target
Before=shutdown.target reboot.target halt.target
BindsTo=network.target
After=network.target

[Service]
Type=idle
ExecStartPre=/usr/sbin/modprobe vhci-hcd

# %i enthält z.B. "192.168.0.231:1-1.2". Wir extrahieren Server und Bus-ID sauber:
ExecStart=/bin/bash -c 'INPUT="%i"; SERVER="${INPUT%%:*}"; BUSID="${INPUT##*:}"; /usr/bin/usbip attach -r "$SERVER" -b "$BUSID"'

# Beim Detach suchen wir nach dem aktiven Port und trennen diesen gezielt
ExecStop=/usr/bin/sh -c '/usr/bin/usbip port | grep "<Port Use in>" | awk "{print \$2}" | tr -d ":" | xargs -r /usr/bin/usbip detach -p'
RemainAfterExit=yes
Restart=on-failure
RestartSec=15s

[Install]
WantedBy=multi-user.target

```

`/etc/systemd/system/usbip-watchdog@.timer`:

```ini
[Unit]
Description=Timer for USB/IP Client Watchdog (%i)

[Timer]
# Der Timer startet 10 Sekunden nach dem Booten
OnBootSec=10s
# Der Timer wird alle 30 Sekunden ausgelöst
OnUnitActiveSec=30s

[Install]
WantedBy=timers.target

```

`/etc/systemd/system/usbip-watchdog@.service`:

```ini
[Unit]
Description=USB/IP Client Watchdog Service (%i)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/usbip-watchdog.sh %i

```

`/usr/local/bin/usbip-watchdog.sh`:

```bash
#!/bin/bash

# Das Argument ist z.B. "192.168.0.231:1-1.2"
INPUT="$1"

# Aufteilen in Server und Bus-ID
SERVER="${INPUT%%:*}"
BUS_ID="${INPUT##*:}"

if [ -z "$SERVER" ] || [ -z "$BUS_ID" ]; then
    echo "WATCHDOG ERROR: Ungültiges Format. Erwartet: SERVER:BUSID (erhalten: $INPUT)"
    exit 1
fi

SERVICE_NAME="usbip-client@$INPUT.service"

# Prüfen, ob das Gerät vorhanden ist
if ! /usr/bin/usbip port | grep -q "/$BUS_ID"; then
    echo "WATCHDOG: USB/IP Gerät $BUS_ID auf Server $SERVER nicht gefunden. Neustart des Services."
    /usr/bin/systemctl restart "$SERVICE_NAME"
    exit 1
fi

# Alles in Ordnung
exit 0
```

Make sure the watchdog script is executable:

```bash
sudo chmod +x /usr/local/bin/usbip-watchdog.sh
```

At first you have to connect to the USB/IP ports you have released on your servers using the `Server:BusID` format.

For Server 1 (e.g., IP `192.168.0.231`, Device `1-1.2`):

```bash
sudo systemctl start usbip-client@192.168.0.231:1-1.2.service
sudo systemctl enable usbip-client@192.168.0.231:1-1.2.service
```

For Server 2 (e.g., IP `192.168.0.245`, Device `1-1.4`):

```bash
sudo systemctl start usbip-client@192.168.0.245:1-1.4.service
sudo systemctl enable usbip-client@192.168.0.245:1-1.4.service
```

After that you have to start the corresponding watchdogs. The watchdog will restart your service if the server is down and restarted a few seconds later.

```bash
sudo systemctl enable --now usbip-watchdog@192.168.0.231:1-1.2.timer
sudo systemctl enable --now usbip-watchdog@192.168.0.245:1-1.4.timer
sudo reboot
```

---

## Conclusion

You can restart both the client and the server. The USB connection via USB/IP should be reestablished each time automatically.
