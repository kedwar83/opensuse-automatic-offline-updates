# Zypper Auto-Update Systemd Services

This project provides a set of systemd services for automating Zypper repository refreshing, update downloading, offline update application, and user notification in case of failures. These services work together to keep your openSUSE or SUSE Linux Enterprise system up-to-date with minimal user intervention.

## Services Overview

1. **Zypper Refresh and Download Updates** (zypper-refresh-download.service)
2. **Zypper Offline Update** (zypper-offline-update.service)
3. **Zypper Update Notification** (zypper-update-notify-failure.service)

## Detailed Service Descriptions and Scripts

### 1. Zypper Refresh and Download Updates (zypper-refresh-download.service)

This service refreshes the Zypper repositories and downloads available updates using the `--download-only` option. It runs once daily, but only after the system has been booted for at least one hour, without user interaction.

#### Script (/usr/local/bin/zypper-refresh-download.sh)

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

#### Service Configuration (/etc/systemd/system/zypper-refresh-download.service)

```ini
[Unit]
Description=Zypper Refresh and Download Updates (Non-interactive)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
TimeoutStartSec=0
ExecStartPre=/bin/sleep 10
ExecStart=/usr/local/bin/zypper-refreshownload.sh
ExecStop=/bin/rm -f /var/run/zypper-update-triggered

[Install]
WantedBy=multi-user.target
```

#### Timer Configuration (/etc/systemd/system/zypper-refreshownload.timer)

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

### 2. Zypper Offline Update (zypper-offline-update.service)

This service applies the downloaded updates during system shutdown, ensuring that the system is up-toate when it next boots.

#### Script (/usr/local/bin/zypper-offline-update.sh)

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

#### Service Configuration (/etc/systemd/system/zypper-offline-update.service)

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

### 3. Zypper Update Notification (zypper-update-notify-failure.service)

This service checks if either the refresh and download service or the offline update service has failed. It waits for a user to log in and then sends a desktop notification if a failure occurred.

#### Script (/usr/local/bin/zypper-update-notifyelayed.sh)

```bash
#!/bin/bash

# Function to check if a user is logged in
user_logged_in() {
    who | grep -q '('
}

# Wait for user login (with a maximum wait time of 1 hour)
timeout=3600
counter=0
while ! user_logged_in && [ $counter -lt $timeout ]; do
    sleep 10
    counter=$((counter + 10))
done

# If no user logged in after timeout, exit
if [ $counter -ge $timeout ]; then
    echo "No user logged in after 1 hour. Exiting." | systemd-cat -t zypper-auto-update -p info
    exit 0
fi

# Wait an additional 5 minutes
sleep 300

# Get the first logged-in user
user=$(who | grep '(' | head -1 | awk '{print $1}')

# Check if either service failed
if systemctl is-failed --quiet zypper-refresh-download.service || \
   systemctl is-failed --quiet zypper-offline-update.service; then
    sudo -u $user DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u $user)/bus notify-send "Zypper Update Failed" "One of the Zypper services has failed. Please check the logs."
fi
```

#### Service Configuration (/etc/systemd/system/zypper-update-notify-failure.service)

```ini
[Unit]
Description=Notify if Zypper Update Services Failed (Delayed)
After=zypper-refresh-download.service zypper-offline-update.service

[Service]
Type=simple
ExecStart=/usr/local/bin/zypper-update-notify-delayed.sh

[Install]
WantedBy=multi-user.target
```

## Installation and Usage

1. Clone this repository or download the scripts and service files.

2. Save the scripts to their respective locations:
   ```
   sudo cp zypper-refresh-download.sh zypper-offline-update.sh zypper-update-notify-delayed.sh /usr/local/bin/
   ```

3. Make the scripts executable:
   ```
   sudo chmod +x /usr/local/bin/zypper-refresh-download.sh /usr/local/bin/zypper-offline-update.sh /usr/local/bin/zypper-update-notify-delayed.sh
   ```

4. Copy the service files to the systemd directory:
   ```
   sudo cp zypper-refresh-download.service zypper-offline-update.service zypper-update-notify-failure.service /etc/systemd/system/
   ```

5. Copy the timer file:
   ```
   sudo cp zypper-refresh-download.timer /etc/systemd/system/
   ```

6. Reload the systemd daemon:
   ```
   sudo systemctl daemon-reload
   ```

7. Enable and start the services:
   ```
   sudo systemctl enable --now zypper-refresh-download.timer zypper-offline-update.service zypper-update-notify-failure.service
   ```

## Troubleshooting

If you encounter issues:

1. Check the status of the services:
   ```
   sudo systemctl status zypper-refresh-download.service
   sudo systemctl status zypper-offline-update.service
   sudo systemctl status zypper-update-notify-failure.service
   ```

2. View the system logs:
   ```
   sudo journalctl -u zypper-refresh-download
   sudo journalctl -u zypper-offline-update
   sudo journalctl -u zypper-update-notify-failure
   ```

3. Ensure that Zypper is working correctly by running basic commands manually:
   ```
   sudo zypper refresh
   sudo zypper dup --dry-run
   ```

## Configuration

You can modify the behavior of these services by editing the script files in `/usr/local/bin/` or the service files in `/etc/systemd/system/`. After making changes, remember to reload the systemd daemon and restart the affected services.

## Contributing

Contributions to improve these services are welcome. Please submit pull requests or open issues on the project's GitHub repository.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
