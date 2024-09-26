# Zypper Upgrade Notification Services

This repository contains systemd services designed to run `zypper dist-upgrade` during system shutdown and to notify the user of the success or failure of the upgrade upon the next boot.

## Overview

- **zypper-upgrade.service**: Executes `zypper dist-upgrade` when the system is shutting down. If the upgrade fails, it creates a marker file to indicate the failure.
- **upgrade-notification.service**: Sends a desktop notification upon boot, indicating whether the last `zypper dist-upgrade` was successful or failed.

## Prerequisites

- A Linux distribution that uses `systemd`.
- `zypper` package manager installed (typically found in openSUSE).
- `notify-send` command available (part of the `libnotify` package).

## Installation

1. **Create Service Files**:

   Create the following service files in the `/etc/systemd/system/` directory.

   ### 1. zypper-upgrade.service

   Create the file with the command:

   ```bash
   sudo vim /etc/systemd/system/zypper-upgrade.service
   ```
Add the following content:

```ini

[Unit]
Description=Run Zypper Dist-Upgrade at Shutdown
DefaultDependencies=no
Before=shutdown.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '/usr/bin/zypper dist-upgrade -y -l --auto-agree-with-product-licenses --no-recommends || touch /tmp/zypper-upgrade-failed'
RemainAfterExit=yes

[Install]
WantedBy=halt.target reboot.target
```
Create the file with the command:

```bash

sudo vim /etc/systemd/system/upgrade-notification.service
```
Add the following content:

```ini

[Unit]
Description=Notify if Zypper Upgrade Failed
After=graphical.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '\
if [ -f /tmp/zypper-upgrade-failed ]; then \
    notify-send "Zypper Upgrade Failed" "The last zypper dist-upgrade did not complete successfully."; \
    rm /tmp/zypper-upgrade-failed; \
fi'

[Install]
WantedBy=default.target
```

Reload Systemd Daemon:

After creating the service files, reload the systemd daemon to recognize the new services:

```bash
sudo systemctl daemon-reload
```
Enable the Services:

Enable both services so that they start as required:

```bash

    sudo systemctl enable zypper-upgrade.service
    sudo systemctl enable upgrade-notification.service
```
