# WiFi Watchdog for MainsailOS (Debian / NetworkManager)

This setup monitors WiFi connectivity every 30 seconds and automatically reconnects if the connection drops. It also logs all activity to a file.

---

## Overview

- Checks connectivity via `ping`  
- Uses `nmcli` (NetworkManager) to reconnect WiFi  
- Runs every **30 seconds** via `systemd timer`  
- Writes logs to:  
`/home/pi/printer_data/logs/wifi_watchdog.log`

---

## 1. Create the script

```bash
mkdir -p /home/pi/wifi_checker
nano /home/pi/wifi_checker/wifi_watchdog.sh
```

Paste:

```
#!/bin/bash

TARGET="192.168.1.1"
IFACE="wlan0"
LOG="/home/pi/printer_data/logs/wifi_watchdog.log"

DATE="$(date '+%Y-%m-%d %H:%M:%S')"

mkdir -p "$(dirname "$LOG")"

if ping -I "$IFACE" -c 2 -W 3 "$TARGET" > /dev/null 2>&1; then
    echo "$DATE OK: $IFACE can reach $TARGET" >> "$LOG"
    exit 0
fi

echo "$DATE FAIL: Ping to $TARGET failed on $IFACE; cycling Wi-Fi" >> "$LOG"
logger -t checkwifi "Ping to $TARGET failed on $IFACE; cycling Wi-Fi"

nmcli device disconnect "$IFACE" || true
sleep 5
nmcli device connect "$IFACE" || true

sleep 10

if ping -I "$IFACE" -c 2 -W 3 "$TARGET" > /dev/null 2>&1; then
    echo "$DATE RECOVERY: $IFACE reconnected successfully" >> "$LOG"
else
    echo "$DATE RECOVERY_FAILED: $IFACE still cannot reach $TARGET" >> "$LOG"
fi
```

Make it executable:  
`chmod +x /home/pi/wifi_checker/wifi_watchdog.sh`

---

## 2. Create systemd service

```bash
sudo nano /etc/systemd/system/wifi_watchdog.service
```

Paste:

```
[Unit]
Description=WiFi watchdog

[Service]
Type=oneshot
ExecStart=/home/pi/wifi_checker/wifi_watchdog.sh
```

---

## 3. Create systemd timer (30 sec interval)

```bash
sudo nano /etc/systemd/system/wifi_watchdog.timer
```

Paste:

```
[Unit]
Description=Run WiFi watchdog every 30 seconds

[Timer]
OnBootSec=30
OnUnitActiveSec=30
AccuracySec=1
Unit=wifi_watchdog.service

[Install]
WantedBy=timers.target
```

---

## 4. Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wifi_watchdog.timer
```

---

## 5. Verify it is running

```bash
systemctl status wifi_watchdog.timer
systemctl list-timers --all | grep -i wifi
```

---

## 6. Test manually

```bash
sudo systemctl start wifi_watchdog.service
```

---

## 7. View logs

File log:

```bash
tail -f /home/pi/printer_data/logs/wifi_watchdog.log
```

System log:

```bash
journalctl -t checkwifi -f
```

---

## 8. Check for existing/conflicting setups

```bash
systemctl list-timers --all | grep -i wifi
systemctl list-units --all | grep -i wifi
crontab -l | grep -i wifi
sudo crontab -l | grep -i wifi
```

---

## Notes

- Assumes NetworkManager is in use (`nmcli`)  
- Default interface: `wlan0`  
- Default gateway target: `192.168.1.1` (adjust if needed)  
- `systemd timer` is used instead of cron for sub-minute execution