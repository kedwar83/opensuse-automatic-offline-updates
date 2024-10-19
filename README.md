# Zypper Auto-Update System

A systemd-based automatic update system for openSUSE/SUSE systems that safely downloads and applies updates while notifying users of any failures. The system is designed to be resilient to system sleep/suspend states and handles interruptions gracefully.

## Features

- Automatic daily download of updates
- Resilient to system sleep/suspend interruptions
- Resume capability for interrupted downloads
- Safe offline installation during system shutdown
- Desktop notifications for update failures
- Comprehensive logging
- Package-by-package download tracking
- Lock file management to prevent conflicts

## Components

### 1. Update Downloader (zypper-refresh-download.sh)

Downloads package updates with sleep state handling and resume capability.

```bash
#!/bin/bash
# /usr/local/bin/zypper-refresh-download.sh

LOCK_FILE="/var/run/zypper-download.lock"
TRIGGER_FILE="/var/run/zypper-update-triggered"
FAILED_FILE="/var/run/zypper-update-failed"
RESUME_FILE="/var/run/zypper-download-resume"

cleanup() {
    local exit_code=$?
    # If we're exiting due to a signal (like sleep), save progress
    if [ $exit_code -ne 0 ] && [ -f "$LOCK_FILE" ]; then
        echo "Saving download state for resume" | systemd-cat -t zypper-auto-update -p info
        if [ -f "$TEMP_DIR/updates.list" ]; then
            # Save remaining packages to resume file
            tail -n +$((CURRENT_PACKAGE + 1)) "$TEMP_DIR/updates.list" > "$RESUME_FILE"
        fi
        /usr/bin/touch "$FAILED_FILE"
    fi
    rm -f "$LOCK_FILE"
    rm -rf "$TEMP_DIR"
}

# Set up traps for cleanup
trap cleanup EXIT INT TERM

# Check if another instance is running
if [ -f "$LOCK_FILE" ]; then
    echo "Another download process is running or was interrupted" | systemd-cat -t zypper-auto-update -p warning
    exit 1
fi

# Create lock file and temp directory
touch "$LOCK_FILE"
TEMP_DIR=$(mktemp -d)
cd "$TEMP_DIR" || exit 1

# Check if we're resuming a previous download
if [ -f "$RESUME_FILE" ]; then
    echo "Resuming previous download" | systemd-cat -t zypper-auto-update -p info
    cp "$RESUME_FILE" "$TEMP_DIR/updates.list"
    rm -f "$RESUME_FILE"
else
    # Fresh start - refresh repositories
    if ! /usr/bin/zypper refresh; then
        echo "Repository refresh failed" | systemd-cat -t zypper-auto-update -p err
        exit 1
    fi

    # Get list of packages to update
    if /usr/bin/zypper --quiet patches-check && /usr/bin/zypper --quiet list-updates; then
        /usr/bin/zypper list-updates --no-refresh | grep "|" | tail -n +4 | cut -d"|" -f2 > "$TEMP_DIR/updates.list"
    else
        echo "No updates available" | systemd-cat -t zypper-auto-update -p info
        rm -f "$FAILED_FILE"
        exit 0
    fi
fi

# Download packages in smaller batches
BATCH_SIZE=5
TOTAL_FAILED=0
CURRENT_PACKAGE=0

while IFS= read -r package; do
    CURRENT_PACKAGE=$((CURRENT_PACKAGE + 1))
    
    if ! /usr/bin/zypper download "$package"; then
        TOTAL_FAILED=$((TOTAL_FAILED + 1))
        echo "Failed to download package: $package" | systemd-cat -t zypper-auto-update -p warning
        sleep 2  # Brief pause before next package
    else
        echo "Successfully downloaded package: $package" | systemd-cat -t zypper-auto-update -p info
    fi
done < "$TEMP_DIR/updates.list"

if [ $TOTAL_FAILED -eq 0 ]; then
    /usr/bin/touch "$TRIGGER_FILE"
    echo "Download completed successfully" | systemd-cat -t zypper-auto-update -p info
    rm -f "$FAILED_FILE"
    rm -f "$RESUME_FILE"
    exit 0
fi

# If we got here, something went wrong
/usr/bin/touch "$FAILED_FILE"
echo "Download process failed" | systemd-cat -t zypper-auto-update -p err
exit 1
```

### 2. Offline Update Script (zypper-offline-update.sh)

Applies downloaded updates during system shutdown.

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

### 3. Update Notification Script (zypper-update-notify.sh)

Monitors for update failures and notifies users.

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

## SystemD Service Files

### Download Service (zypper-refresh-download.service)

```ini
[Unit]
Description=Zypper Refresh and Download Updates (Non-interactive)
Wants=network-online.target
After=network-online.target
Before=sleep.target
RefuseManualStart=no
RefuseManualStop=no
StopWhenUnneeded=yes

[Service]
Type=oneshot
TimeoutStartSec=3600
TimeoutStopSec=300
KillMode=mixed
KillSignal=SIGTERM

ExecStartPre=/bin/rm -f /var/run/zypper-download.lock
ExecStart=/usr/local/bin/zypper-refresh-download.sh
ExecStop=/bin/sh -c '/bin/kill -TERM $MAINPID; sleep 2'
ExecStopPost=/bin/rm -f /var/run/zypper-download.lock

Restart=on-failure
RestartSec=60s
StartLimitInterval=24h
StartLimitBurst=3

[Install]
WantedBy=multi-user.target
```

### Download Timer (zypper-refresh-download.timer)

```ini
[Unit]
Description=Run Zypper Refresh and Download Updates daily

[Timer]
OnBootSec=10m
OnUnitActiveSec=24h
RandomizedDelaySec=2h

[Install]
WantedBy=timers.target
```

### Offline Update Service (zypper-offline-update.service)

```ini
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

### Notification Service (zypper-update-notify-failure.service)

```ini
[Unit]
Description=Notify if Zypper Update Failed
After=graphical-session.target

[Service]
Type=simple
ExecStart=/usr/local/bin/zypper-update-notify.sh

[Install]
WantedBy=default.target
```

### Notification Path Unit (zypper-update-notify-failure.path)

```ini
[Unit]
Description=Watch for Zypper update failure flag

[Path]
PathExists=/var/run/zypper-update-failed
Unit=zypper-update-notify-failure.service

[Install]
WantedBy=multi-user.target
```

## Installation

1. Create the necessary directories:
```bash
sudo mkdir -p /usr/local/bin
```

2. Create all script files with the content provided above and make them executable:
```bash
sudo cp zypper-refresh-download.sh /usr/local/bin/
sudo cp zypper-offline-update.sh /usr/local/bin/
sudo cp zypper-update-notify.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/zypper-*.sh
```

3. Create the systemd service files:
```bash
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

1. **Update Download Process**
   - Timer triggers download service daily
   - Updates are downloaded one package at a time
   - Progress is saved if interrupted by sleep/suspend
   - Trigger file created when updates are ready
   - Failures are logged and flagged for notification

2. **Sleep/Suspend Handling**
   - When system sleeps, current progress is saved
   - Download resumes from last point after wake
   - Lock files prevent multiple instances
   - Proper cleanup ensures system consistency

3. **Update Installation**
   - Updates installed during system shutdown
   - Ensures no services are running during update
   - Process is automatic and requires no user intervention

4. **Notification System**
   - Monitors for update failures using path unit
   - Displays desktop notifications on failures
   - Automatically cleans up notification flags

## Troubleshooting

Check service status:
```bash
# Check download service status
sudo systemctl status zypper-refresh-download.service

# Check offline update service status
sudo systemctl status zypper-offline-update.service

# Check notification path unit status
sudo systemctl status zypper-update-notify-failure.path
```

View logs:
```bash
# View download service logs
sudo journalctl -u zypper-refresh-download

# View offline update logs
sudo journalctl -u zypper-offline-update

# View notification service logs
sudo journalctl -u zypper-update-notify-failure
```

## Common Issues

1. **Downloads Failing During Sleep**
   - Check `/var/run/zypper-download-resume` for interrupted downloads
   - View logs to see which packages failed
   - Manually remove lock file if necessary: `sudo rm /var/run/zypper-download.lock`

2. **Updates Not Installing**
   - Verify trigger file exists: `ls -l /var/run/zypper-update-triggered`
   - Check offline update logs for errors
   - Ensure offline update service is enabled

3. **No Notifications**
   - Verify desktop environment is running
   - Check notification service logs
   - Ensure the user has proper permissions

## License

This software is released under the MIT License.
