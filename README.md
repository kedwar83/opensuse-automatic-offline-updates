
# Transactional Update Error Checker

This setup creates two systemd services that manage updates on shutdown and notify the user of any errors during the next boot.

- The first service runs `transactional-update` during shutdown.
- The second service checks for errors during the next boot and notifies the user if any issues occurred.

## Installation Steps

### 1. Create the Transactional Update Shutdown Service

This service runs `transactional-update` during shutdown and logs its output.

1. Create the systemd service file:
```bash
sudo nano /etc/systemd/system/transactional-update-shutdown.service
```
Add the following content:

```ini

[Unit]
Description=Run transactional-update on shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 5
ExecStart=/usr/sbin/transactional-update up
StandardOutput=journal
StandardError=journal
TimeoutStopSec=30  

[Install]
WantedBy=halt.target reboot.target shutdown.target

```
Enable the service:

```bash

    sudo systemctl enable transactional-update-shutdown.service
```
2. Create the Check Update Error Service

This service runs on boot to check if the previous transactional-update resulted in any errors and notifies the user if so.

Create the systemd service file:

```bash
sudo nano /etc/systemd/system/transactional-update-check.service
```
Add the following content:

```ini

[Unit]
Description=Check for transactional-update errors on boot
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/check-transactional-update-errors.sh

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash

    sudo systemctl enable transactional-update-check.service
```
3. Create the Error Checking Script

This script checks the log file for errors and notifies the user via desktop notification (or a message to the terminal) if any errors are found.

Create the script:

```bash

sudo nano /usr/local/bin/check-transactional-update-errors.sh
```
Add the following content:

```bash

#!/bin/bash

LOG_FILE="/var/log/transactional-update.log"

# Check if the log file contains errors (adjust the keyword based on the error output)
if grep -qi "error" "$LOG_FILE"; then
    # Show an error message using the wall command (for terminal output) or notify-send for graphical sessions
    if command -v notify-send > /dev/null; then
        notify-send "Transactional Update" "Errors were detected during the last transactional update. Check the log at /var/log/transactional-update.log."
    else
        wall "Errors were detected during the last transactional update. Check the log at /var/log/transactional-update.log."
    fi
fi
```
Make the script executable:

```bash

    sudo chmod +x /usr/local/bin/check-transactional-update-errors.sh
```
