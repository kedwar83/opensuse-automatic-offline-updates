# Zypper Auto-Update System

A systemd-based automatic update system for openSUSE/SUSE systems that safely downloads and applies updates while notifying users of any failures.

## Overview

This system consists of three main components:

1. **Update Downloader** (`zypper-refresh-download`)
   - Downloads package updates daily without installing them
   - Creates a trigger file when updates are ready to install

2. **Offline Updater** (`zypper-offline-update`)
   - Applies downloaded updates during system shutdown
   - Ensures system consistency by updating when services are inactive

3. **Failure Notifier** (`zypper-update-notify-failure`)
   - Monitors for update failures
   - Displays desktop notifications to logged-in users when issues occur

## Scripts and Configuration Files

### 1. Update Download Script
```bash
#!/bin/bash
# /usr/local/bin/zypper-refresh-download.sh

# Refresh repositories and download updates (non-interactive)
if /usr/bin/zypper refresh && /usr/bin/zypper dup -y --no-recommends --download-only; then
    /usr/bin/touch /var/run/zypper-update-triggered
    exit 0
else
    # Create a failure flag file that the notification service will check
    /usr/bin/touch /var/run/zypper-update-failed
    exit 1
fi
```

### 2. Offline Update Script
```bash
#!/bin/bash
# /usr/local/bin/zypper-offline-update.sh

# Check if updates were downloaded successfully
if [ -f /var/run/zypper-update-triggered ]; then
    # Apply updates
    if /usr/bin/zypper dup -y --no-recommends; then
        echo "Offline update applied successfully" | systemd-cat -t zypper-auto-update -p info
    else
        echo "Offline update failed" | systemd-cat -t zypper-auto-update -p err
    fi

    # Remove the trigger file
    rm /var/run/zypper-update-triggered
else
    echo "No updates to apply or download was incomplete" | systemd-cat -t zypper-auto-update -p info
fi

# Ensure the service exits cleanly
exit 0
```

### 3. Update Notification Script
```bash
#!/bin/bash
# /usr/local/bin/zypper-update-notify.sh

# Wait a short time for the desktop environment to be fully loaded
sleep 30

# Get the first logged-in user
user=$(who | grep '(' | head -1 | awk '{print $1}')

if [ -n "$user" ] && [ -f /var/run/zypper-update-failed ]; then
    sudo -u $user DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u $user)/bus notify-send "Zypper Update Failed" "The Zypper update download has failed. Please check the logs."
    rm -f /var/run/zypper-update-failed
fi
```

### 4. Systemd Service Files

#### Download Service
```ini
# /etc/systemd/system/zypper-refresh-download.service
[Unit]
Description=Zypper Refresh and Download Updates (Non-interactive)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/zypper-refresh-download.sh
ExecStop=/bin/rm -f /var/run/zypper-update-triggered /var/run/zypper-update-failed

[Install]
WantedBy=multi-user.target
```

#### Download Timer
```ini
# /etc/systemd/system/zypper-refresh-download.timer
[Unit]
Description=Run Zypper Refresh and Download Updates daily

[Timer]
OnBootSec=10m
OnUnitActiveSec=24h
RandomizedDelaySec=2h

[Install]
WantedBy=timers.target
```

#### Offline Update Service
```ini
# /etc/systemd/system/zypper-offline-update.service
[Unit]
Description=Zypper Offline Update
DefaultDependencies=no
Conflicts=shutdown.target
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/zypper-offline-update.sh

[Install]
WantedBy=shutdown.target
```

#### Notification Service and Path Unit
```ini
# /etc/systemd/system/zypper-update-notify-failure.service
[Unit]
Description=Notify if Zypper Update Failed
After=graphical-session.target

[Service]
Type=simple
ExecStart=/usr/local/bin/zypper-update-notify.sh

[Install]
WantedBy=default.target

# /etc/systemd/system/zypper-update-notify-failure.path
[Unit]
Description=Watch for Zypper update failure flag

[Path]
PathExists=/var/run/zypper-update-failed
Unit=zypper-update-notify-failure.service

[Install]
WantedBy=multi-user.target
```

## Installation

1. Create the script files:
```bash
sudo mkdir -p /usr/local/bin
```

2. Copy the scripts and make them executable:
```bash
# Copy the scripts (after creating them with the content above)
sudo cp zypper-refresh-download.sh /usr/local/bin/
sudo cp zypper-offline-update.sh /usr/local/bin/
sudo cp zypper-update-notify.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/zypper-*.sh
```

3. Create and copy the systemd service files:
```bash
# Copy the service files (after creating them with the content above)
sudo cp zypper-refresh-download.service /etc/systemd/system/
sudo cp zypper-refresh-download.timer /etc/systemd/system/
sudo cp zypper-offline-update.service /etc/systemd/system/
sudo cp zypper-update-notify-failure.service /etc/systemd/system/
sudo cp zypper-update-notify-failure.path /etc/systemd/system/
```

4. Reload systemd and enable the services:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zypper-refresh-download.timer
sudo systemctl enable zypper-offline-update.service
sudo systemctl enable --now zypper-update-notify-failure.path
```

## How It Works

### Update Download Process
- The timer triggers the download service once daily
- Updates are downloaded but not installed
- A trigger file is created when updates are ready
- Failures are logged and flagged for notification

### Update Installation Process
- Updates are installed during system shutdown
- This ensures no services are running during update
- The process is automatic and requires no user intervention

### Notification System
- Monitors for update failures using a path unit
- Displays desktop notifications when users log in if failures occurred
- Automatically cleans up notification flags after displaying

## Troubleshooting

### Checking Service Status
```bash
# Check download service status
sudo systemctl status zypper-refresh-download.service

# Check offline update service status
sudo systemctl status zypper-offline-update.service

# Check notification path unit status
sudo systemctl status zypper-update-notify-failure.path
```

### Viewing Logs
```bash
# View download service logs
sudo journalctl -u zypper-refresh-download

# View offline update logs
sudo journalctl -u zypper-offline-update

# View notification service logs
sudo journalctl -u zypper-update-notify-failure
```

### Common Issues

1. **Updates Not Downloading**
   - Check network connectivity
   - Verify repository access: `sudo zypper refresh`
   - Check service logs for specific errors

2. **No Notifications Appearing**
   - Ensure notification daemon is running
   - Check if DISPLAY variable is set correctly
   - Verify user has proper permissions

3. **Updates Not Installing**
   - Check if trigger file exists: `/var/run/zypper-update-triggered`
   - Verify offline update service is enabled
   - Check shutdown logs for errors

## Security Considerations

- Scripts run with root privileges
- Notification system safely drops privileges when displaying to user
- No interactive prompts during update process
- Failure flags stored in `/var/run` for security

## Contributing

Contributions are welcome! Please submit pull requests or open issues for any bugs or feature requests.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
