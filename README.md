## Services

1. **zypper-upgrade.service**: Executes `zypper refresh` and `zypper dist-upgrade` during system shutdown.
2. **upgrade-notification.service**: Sends a desktop notification upon boot about the upgrade status.

## Prerequisites

- openSUSE operating system
- `notify-send` command (part of the `libnotify` package)

## Installation

1. Create service files in `/etc/systemd/system/`:

   **zypper-upgrade.service**:
   ```ini
   [Unit]
[Unit]
Description=Run Zypper Refresh and Dist-Upgrade at Startup
DefaultDependencies=no
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '\
if nm-online -q; then \
    if ! /usr/bin/zypper refresh; then \
        echo "Zypper refresh failed" > /tmp/zypper-refresh-failed; \
        exit 1; \
    fi && \
    if ! /usr/bin/zypper dist-upgrade -y --auto-agree-with-product-licenses --no-recommends; then \
        echo "Zypper dist-upgrade failed" > /tmp/zypper-upgrade-failed; \
        exit 1; \
    fi; \
else \
    echo "No internet connection, skipping updates." > /tmp/no-internet; \
fi'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

   ```

   **upgrade-notification.service**:
   ```ini
   [Unit]
   Description=Notify if Zypper Refresh or Upgrade Failed
   After=graphical.target
   Wants=graphical.target

   [Service]
   Type=oneshot
   ExecStart=/bin/bash -c 'if [ -f /tmp/zypper-refresh-failed ]; then \
       notify-send "Zypper Refresh Failed" "$(cat /tmp/zypper-refresh-failed). Manual intervention may be necessary."; \
       rm /tmp/zypper-refresh-failed; \
   elif [ -f /tmp/zypper-upgrade-failed ]; then \
       notify-send "Zypper Upgrade Failed" "$(cat /tmp/zypper-upgrade-failed)"; \
       rm /tmp/zypper-upgrade-failed; \
   fi'

   [Install]
   WantedBy=default.target
   ```

2. Reload systemd daemon:
   ```
   sudo systemctl daemon-reload
   ```

3. Enable services:
   ```
   sudo systemctl enable zypper-upgrade.service
   sudo systemctl enable upgrade-notification.service
   ```

## Functionality

- `zypper-upgrade.service` runs `zypper refresh` followed by `zypper dist-upgrade` during system shutdown.
- If `zypper refresh` fails, it creates `/tmp/zypper-refresh-failed`.
- If `zypper dist-upgrade` fails, it creates `/tmp/zypper-upgrade-failed`.
- `upgrade-notification.service` checks for these files and sends notification on failure.


## Troubleshooting

Check service status:
```
sudo systemctl status zypper-upgrade.service
sudo systemctl status upgrade-notification.service
```

View logs:
```
journalctl -u zypper-upgrade.service
journalctl -u upgrade-notification.service
```
