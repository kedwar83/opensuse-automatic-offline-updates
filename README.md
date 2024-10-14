# Zypper Auto-Update Systemd Services

This project provides a set of systemd services for automating Zypper repository refreshing, update downloading, offline update application, and user notification in case of failures. These services work together to keep your openSUSE or SUSE Linux Enterprise system up-to-date with minimal user intervention while addressing potential issues with system sleep and incomplete downloads.

## Services Overview

1. Zypper Refresh and Download Updates (zypper-refresh-download.service)
2. Zypper Offline Update (zypper-offline-update.service)
3. Zypper Update Notification (zypper-update-notify-failure.service)

## Detailed Service Descriptions

### 1. Zypper Refresh and Download Updates (zypper-refresh-download.service)

This service refreshes the Zypper repositories and downloads available updates using the --download-only option. It runs once daily, but only after the system has been booted for at least one hour, without user interaction.

#### Script (/usr/local/bin/zypper-refresh-download.sh)

```bash
#!/bin/bash

# Refresh repositories
/usr/bin/zypper refresh

# Download updates (non-interactive) without changing recommendations and other specified options
if /usr/bin/zypper dup -y --no-recommends -D --download-only; then
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
ExecStart=/usr/local/bin/zypper-refresh-download.sh
ExecStop=/bin/rm -f /var/run/zypper-update-triggered

[Install]
WantedBy=multi-user.target
```

#### Timer Configuration (/etc/systemd/system/zypper-refresh-download.timer)

```ini
[Unit]
Description=Run Zypper Refresh and Download Updates daily

[Timer]
OnBootSec=1h
OnUnitActiveSec=24h
RandomizedDelaySec=2h

[Install]
WantedBy=timers.target
```

### 2. Zypper Offline Update (zypper-offline-update.service)

This service applies the downloaded updates during system shutdown, ensuring that the system is up-to-date when it next boots.

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

This service checks if either the refresh and download service or the offline update service has failed, and sends a desktop notification if so.

#### Service Configuration (/etc/systemd/system/zypper-update-notify-failure.service)

```ini
[Unit]
Description=Notify if Zypper Update Services Failed
After=zypper-refresh-download.service zypper-offline-update.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c '\
    if systemctl is-failed --quiet zypper-refresh-download.service || \
       systemctl is-failed --quiet zypper-offline-update.service; then \
        /usr/bin/notify-send "Zypper Update Failed" "One of the Zypper services has failed. Please check the logs."; \
    fi'

[Install]
WantedBy=multi-user.target
```

## Installation and Usage

1. Save the scripts to their respective locations (/usr/local/bin/).
2. Make the scripts executable:
   ```
   sudo chmod +x /usr/local/bin/zypper-refresh-download.sh /usr/local/bin/zypper-offline-update.sh
   ```
3. Save the service files to /etc/systemd/system/.
4. Reload the systemd daemon:
   ```
   sudo systemctl daemon-reload
   ```
5. Enable and start the services:
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
   journalctl -u zypper-refresh-download
   journalctl -u zypper-offline-update
   ```
3. Ensure that Zypper is working correctly by running basic commands manually:
   ```
   sudo zypper refresh
   sudo zypper dup --dry-run
   ```

## Sleep and Freeze Prevention

To address potential issues with system freezes during sleep:

1. The download service now creates a flag file only when the download completes successfully.
2. The offline update service checks for this flag file before proceeding with updates.
3. Both services have been modified to ensure they exit cleanly after completion.
4. The offline update service has been moved to run before the shutdown target, reducing the chance of interference with sleep processes.

If you continue to experience freezes during sleep, consider the following:

1. Check system logs immediately after resuming from a frozen state.
2. Monitor system resources during the update process to ensure it's not causing excessive load.
3. Consider adjusting the timing of the update services to run at times when the system is less likely to be put to sleep.

## Contributing

Contributions to improve these services are welcome. Please submit pull requests or open issues on the project's GitHub repository.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
