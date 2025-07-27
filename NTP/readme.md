# Health Check for your NTP-PPS server with Uptime-Kuma:
<img width="1356" height="450" alt="ntp-pps1" src="https://github.com/user-attachments/assets/be822f8b-2cc0-4df7-921c-6d9c6273d801" />


> ## Time Sync Health
> NTP and Chrony are critical for precise time synchronization, especially in self-hosted environments where PPS (pulse-per-second) signals are used to dramatically increase accuracy, often down to the microsecond. However, these tools don’t expose an HTTP or TCP based endpoint that Uptime Kuma can monitor directly. Without a dedicated service to report time sync status (e.g., via chronyc tracking or ntpq -p), Kuma can’t detect if synchronization is failing, drifting, or degraded. Inaccurate time can silently break TLS, invalidate logs (yes you, alloy), corrupt time-series data, or cause cluster nodes to misbehave.  This makes reliable monitoring essential and with this healthcheck endpoint, it helps to ensure your time source remains trusted and stable.

### 1. Uptime Kuma settings (starting here to get the necessary token authenticated webhook)
- New Monitor
- Monitor type **Push**
- Heartbeat interval 180 s (3 min)
- Save
- Get the link provided after saving for the next step

> **Note:**  Set the interval to whatever you like here. A 60s heartbeat lands comfortably inside the 180s window, so Kuma never misses a push even with sporadic timer jitter, but that should be minimal anyway.

### 2: create a script and then save wherever - Example: /usr/local/bin or ~/scripts 
`nano ntp-status.sh`

> **IMPORTANT** you need to remove the **query string** [?status=up&msg=OK&ping=] in the `PUSH_URL` variable as it might get double-referenced in this script otherwise
````
#!/usr/bin/env bash
set -euo pipefail
trap '' SIGPIPE

TIMEOUT=${TIMEOUT:-2}

# this url is the one you got from uptime-kuma - dont forget to remove the `query string` mentioned above
PUSH_URL="https://<uptime.domain.com>/api/push/<token-from-uptime>"

run() { timeout "$TIMEOUT" "$@" 2>/dev/null; }

# 1) Make sure the service is up (tested on debian|ubuntu|server|22.04|24.04)
if ! (systemctl is-active --quiet chronyd 2>/dev/null || systemctl is-active --quiet chrony 2>/dev/null); then
  echo "chrony service not active"
  curl -fsS "${PUSH_URL}?status=down&msg=NTP_desync" >/dev/null
  exit 1
fi

# 2) Does any source show the '*' selection flag as the second character?
sources="$(run chronyc -n sources || true)"
if [[ -n "$sources" ]]; then
  if printf '%s\n' "$sources" | awk 'NR>2 && substr($1,2,1)=="*"{exit 0} END{exit 1}'; then
    echo "OK: chrony synced (sources)"
    curl -fsS "${PUSH_URL}?status=up&msg=OK" >/dev/null
    exit 0
  fi
fi

# 3) Fallback: for PPS/reference-clock cases
tracking="$(run chronyc tracking || true)"
if [[ -z "$tracking" ]]; then
  echo "chronyc not responding (timeout or error)"
  curl -fsS "${PUSH_URL}?status=down&msg=NTP_desync" >/dev/null
  exit 1
fi

stratum="$(awk -F': *' '/Stratum/ {print $2+0}' <<<"$tracking" 2>/dev/null || echo 16)"
leap="$(awk -F': *' '/Leap status/ {print $2}' <<<"$tracking" 2>/dev/null || echo "Unknown")"

if [[ "$stratum" -ne 16 && "$leap" == "Normal" ]]; then
  echo "OK: chrony synced (tracking)"
  curl -fsS "${PUSH_URL}?status=up&msg=OK" >/dev/null
  exit 0
fi

echo "FAIL: not synchronized (* missing; stratum=$stratum, leap=$leap)"
curl -fsS "${PUSH_URL}?status=down&msg=NTP_desync" >/dev/null
exit 1
`````

### 3: make it executable:
`chmod +x ntp-status.sh`

### 3a: make your systemd service and timer:

#### File layout
```
└─ /etc/systemd/system
   ├─ ntp-status.service   # one‑shot runner
   └─ ntp-status.timer     # 60‑second schedule
```
### 4. ntp-status.service
`sudo nano /etc/systemd/system/ntp-status.service`
````
[Unit]
Description=Push Chrony health to Uptime Kuma

[Service]
Type=oneshot
ExecStart=/path/to/script/ntp-status.sh
````

### 5. ntp-status.timer
`sudo nano /etc/systemd/system/ntp-status.timer`

```
[Unit]
Description=Run ntp‑status.service every 60 s

[Timer]
OnBootSec=30s           # first run 30 s after boot
OnUnitActiveSec=60s     # then every 60 s thereafter
AccuracySec=2s          # max jitter added by systemd
Unit=ntp-status.service

[Install]
WantedBy=timers.target
```

- OnUnitActiveSec= creates a fixed loop 
- AccuracySec= keeps drift low 

### 6. Enable & start
- `sudo systemctl daemon-reload`
- `sudo systemctl enable --now ntp-status.timer`

### 7. Verify
`systemctl status ntp-status.timer`


### 8. Feast and be merry! Your precision Stratum 1 NTP-PPS server is being monitored!

> **Thoughts:**
> This will probably work for any of the uptime services currently available such as peekaping, tianji, checkmate, etc....  as long as they can provide a push URL.

