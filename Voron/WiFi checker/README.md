# Wi-Fi watchdog for MainsailOS

The Raspberry Pi uses a custom Wi-Fi watchdog, separate from Sonar. The following matches the installed configuration checked on 16 October 2026.

The watchdog checks `wlan0` approximately every 30 seconds, starting 30 seconds after boot. It checks for an IPv4 address and pings the first default gateway returned by the routing table through `wlan0`. If that check fails, it cycles the interface and restarts NetworkManager (or falls back to dhcpcd/wpa_supplicant if NetworkManager is inactive). Results are appended to `/home/pi/printer_data/logs/wifi_watchdog.log`.

Perform installation and manual testing while the printer is idle: a failed check can restart networking and disconnect the SSH session. These instructions document the existing recovery mechanism; they do not identify or eliminate the underlying cause of Wi-Fi disconnections.

1. Create the script directory and open the script:

```bash
mkdir -p /home/pi/wifi_checker
nano /home/pi/wifi_checker/wifi_watchdog.sh
```

Paste and save:

```bash
#!/usr/bin/env bash

LOG="/home/pi/printer_data/logs/wifi_watchdog.log"
IFACE="wlan0"
DATE="$(date '+%Y-%m-%d %H:%M:%S')"

mkdir -p "$(dirname "$LOG")"

GW="$(ip route | awk '/default/ {print $3; exit}')"

if ip link show "$IFACE" >/dev/null 2>&1 && \
   ip -4 addr show "$IFACE" | grep -q "inet " && \
   ping -I "$IFACE" -c 1 -W 3 "$GW" >/dev/null 2>&1; then
    echo "$DATE OK: WiFi connected on $IFACE, gateway $GW reachable" >> "$LOG"
    exit 0
fi

echo "$DATE FAIL: WiFi appears down. Restarting $IFACE..." >> "$LOG"

ip link set "$IFACE" down
sleep 3
ip link set "$IFACE" up
sleep 5

if systemctl is-active --quiet NetworkManager; then
    systemctl restart NetworkManager
elif systemctl is-active --quiet dhcpcd; then
    systemctl restart dhcpcd
elif systemctl is-active --quiet wpa_supplicant; then
    systemctl restart wpa_supplicant
fi

sleep 10

if ip -4 addr show "$IFACE" | grep -q "inet "; then
    echo "$DATE RECOVERY: $IFACE has IP again" >> "$LOG"
else
    echo "$DATE RECOVERY_FAILED: $IFACE still has no IP" >> "$LOG"
fi
```

2. Make the script executable:

```bash
chmod +x /home/pi/wifi_checker/wifi_watchdog.sh
```

3. Create the systemd service:

```bash
sudo nano /etc/systemd/system/wifi_watchdog.service
```

Paste and save:

```ini
[Unit]
Description=WiFi watchdog

[Service]
Type=oneshot
ExecStart=/home/pi/wifi_checker/wifi_watchdog.sh
```

4. Create the timer:

```bash
sudo nano /etc/systemd/system/wifi_watchdog.timer
```

Paste and save:

```ini
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

5. Check the script syntax, reload systemd and perform one manual test while idle:

```bash
bash -n /home/pi/wifi_checker/wifi_watchdog.sh
sudo systemctl daemon-reload
sudo systemctl start wifi_watchdog.service
journalctl -u wifi_watchdog.service -n 20 --no-pager
tail -n 20 /home/pi/printer_data/logs/wifi_watchdog.log
```

The log should report `OK: WiFi connected` when the check passes. On a failed check, inspect the `FAIL`, `RECOVERY` or `RECOVERY_FAILED` entries. `RECOVERY` means an IPv4 address returned; it does not itself verify gateway connectivity.

6. Enable the timer and verify it:

```bash
sudo systemctl enable --now wifi_watchdog.timer
systemctl status wifi_watchdog.timer --no-pager
systemctl list-timers --all wifi_watchdog.timer
```

The timer should be enabled and active. The service is `Type=oneshot`, so it normally appears inactive between checks; the timer invokes it again. There is no need to enable the service separately.
