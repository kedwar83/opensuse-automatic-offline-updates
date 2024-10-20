# Zypper Automatic Update System

This system provides automated management of system updates for openSUSE/SUSE Linux using zypper. It consists of three main components that work together to safely download and apply updates while keeping users informed of the process.

## Features

- Automated daily download of system updates
- Offline update application during system shutdown
- Desktop notifications for update status and failures
- Non-interactive operation
- Configurable timing with randomized delays
- Safe update practices (no automatic recommendations)

## Components

### 1. Zypper Refresh and Download Service
Downloads updates on a daily schedule without applying them.
- Downloads updates in non-interactive mode
- Runs daily with a randomized delay
- Creates a flag file when updates are ready to apply

#### Script (`/usr/local/bin/zypper-refresh-download.sh`):
```bash
#!/bin/bash

# Refresh repositories
/usr/bin/zypper refresh

# Download updates (non-interactive) without changing recommendations and other specified options
if /usr/bin/zypper dup -y --no-recommends --download-only; then
    # Create a flag file to indicate the update was triggered and completed successfully
    /usr/bin/touch /var/run/zypper-update-triggered
    echo "Update download completed successfully" | systemd-cat -t zypper-auto-update -p info
else
    echo "Update download failed" | systemd-cat -t zypper-auto-update -p err
    exit 1
fi

# Ensure the service exits cleanly
exit 0
```

#### Service Configuration (`/etc/systemd/system/zypper-refresh-download.service`):
```ini
[Unit]
Description=Zypper Refresh and Download Updates (Non-interactive)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
TimeoutStartSec=0
ExecStartPre=/bin/sleep 10
ExecStart=/usr/local/bin/zypper-refresh-download.sh
ExecStop=/bin/rm -f /var/run/zypper-update-triggered

[Install]
WantedBy=multi-user.target
```

#### Timer Configuration (`/etc/systemd/system/zypper-refresh-download.timer`):
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

### 2. Zypper Offline Update Service
Applies downloaded updates during system shutdown.
- Ensures updates are applied when the system is in a clean state
- Prevents interruption of running applications
- Only runs if updates were successfully downloaded

#### Script (`/usr/local/bin/zypper-offline-update.sh`):
```bash
#!/bin/bash

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

#### Service Configuration (`/etc/systemd/system/zypper-offline-update.service`):
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

### 3. Update Notification Service
Notifies users about update status upon login.
- Provides desktop notifications about update success or failure
- Triggers automatically when users log in
- Uses the system's native notification system

#### Script (`/usr/local/bin/zypper-update-notify.sh`):
```bash
#!/bin/bash

# Get the current user
user=$PAM_USER

# Ensure we have a user
if [ -z "$user" ]; then
    echo "No user specified" | systemd-cat -t zypper-auto-update -p err
    exit 1
fi

# Wait a few seconds for the session to be fully initialized
sleep 5

# Check if either service failed
if systemctl is-failed --quiet zypper-refresh-download.service || \
   systemctl is-failed --quiet zypper-offline-update.service; then
    sudo -u $user DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u $user)/bus \
    notify-send -u critical "Zypper Update Failed" "One of the Zypper services has failed. Please check the logs."
else
    # Optionally notify of success
    sudo -u $user DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u $user)/bus \
    notify-send "Zypper Update Status" "System updates are working normally."
fi

exit 0
```

#### PAM Configuration (`/etc/pam.d/login-notification`):
```
session optional pam_exec.so /usr/local/bin/zypper-update-notify.sh
```

#### Service Configuration (`/etc/systemd/system/zypper-update-notify.service`):
```ini
[Unit]
Description=Zypper Update Notification Service
After=zypper-refresh-download.service zypper-offline-update.service

[Service]
Type=simple
ExecStart=/usr/bin/true

[Install]
WantedBy=multi-user.target
```

## Installation

1. Create all scripts:
```bash
# Create script directories if they don't exist
sudo mkdir -p /usr/local/bin

# Create all scripts with the content shown above
sudo nano /usr/local/bin/zypper-refresh-download.sh
sudo nano /usr/local/bin/zypper-offline-update.sh
sudo nano /usr/local/bin/zypper-update-notify.sh

# Make scripts executable
sudo chmod +x /usr/local/bin/zypper-refresh-download.sh
sudo chmod +x /usr/local/bin/zypper-offline-update.sh
sudo chmod +x /usr/local/bin/zypper-update-notify.sh
```

2. Create systemd service files:
```bash
# Create service files with the content shown above
sudo nano /etc/systemd/system/zypper-refresh-download.service
sudo nano /etc/systemd/system/zypper-refresh-download.timer
sudo nano /etc/systemd/system/zypper-offline-update.service
sudo nano /etc/systemd/system/zypper-update-notify.service
```

3. Create PAM configuration:
```bash
# Create PAM configuration file
sudo nano /etc/pam.d/login-notification
```

4. Enable and start the services:
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable services
sudo systemctl enable zypper-refresh-download.timer
sudo systemctl enable zypper-offline-update.service
sudo systemctl enable zypper-update-notify.service

# Start the timer
sudo systemctl start zypper-refresh-download.timer
```

## Configuration

### Update Schedule
The default schedule downloads updates daily with a 2-hour random delay. To modify this, edit the timer configuration shown above in the Components section.

### Update Options
The update process uses `--no-recommends` by default. To modify update options, edit the respective scripts shown above in the Components section.

## Troubleshooting

### Checking Service Status
```bash
# Check timer status
systemctl status zypper-refresh-download.timer

# Check download service status
systemctl status zypper-refresh-download.service

# Check offline update service status
systemctl status zypper-offline-update.service

# Check notification service status
systemctl status zypper-update-notify.service
```

### Viewing Logs
```bash
# View all related logs
journalctl -t zypper-auto-update

# View specific service logs
journalctl -u zypper-refresh-download.service
journalctl -u zypper-offline-update.service
journalctl -u zypper-update-notify.service
```

## Security Considerations

- Scripts run with root privileges through systemd
- Update downloads are separate from installation for safety
- Updates are applied during shutdown to prevent service interruption
- Notifications are delivered safely to user sessions

## Contributing

Feel free to submit issues and pull requests for improvements to the system.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
