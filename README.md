# Zypper Auto-Update Systemd Services

This project provides a set of `systemd` services for automating Zypper repository refreshing, update downloading, offline update application, and user notification in case of failures. These services work together to keep your openSUSE or SUSE Linux Enterprise system up-to-date with minimal user intervention.

## Services Overview

1. **Zypper Refresh and Download Updates** (`zypper-refresh-download.service`)
2. **Zypper Offline Update** (`zypper-offline-update.service`)
3. **Zypper Update Notification** (`zypper-update-notify-failure.service`)

## Detailed Service Descriptions

### 1. Zypper Refresh and Download Updates (`zypper-refresh-download.service`)

This service refreshes the Zypper repositories and downloads available updates using the `--download-only` option. It runs daily without user interaction.

#### Script (`/usr/local/bin/zypper-refresh-download.sh`)

```bash
#!/bin/bash

# Refresh repositories
/usr/bin/zypper --non-interactive refresh

# Download updates (non-interactive) without changing recommendations and other specified options
/usr/bin/zypper --non-interactive dup -y --no-recommends --allow-downgrade --allow-name-change --allow-arch-change --allow-vendor-change -D --download-only

# Create a flag file to indicate the update was triggered
/usr/bin/touch /var/run/zypper-update-triggered
```

#### Service Configuration (`/etc/systemd/system/zypper-refresh-download.service`)

```ini
[Unit]
Description=Zypper Refresh and Download Updates (Non-interactive)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
TimeoutStartSec=0
ExecStart=/usr/local/bin/zypper-refresh-download.sh

[Install]
WantedBy=multi-user.target
```

### 2. Zypper Offline Update (`zypper-offline-update.service`)

This service applies the downloaded updates during system shutdown, ensuring that the system is up-to-date when it next boots.

#### Script (`/usr/local/bin/zypper-offline-update.sh`)

```bash
#!/bin/bash

# Check if updates were downloaded
if [ -f /var/run/zypper-update-triggered ]; then
    # Apply updates
    /usr/bin/zypper --non-interactive dup -y --no-recommends --allow-downgrade --allow-name-change --allow-arch-change --allow-vendor-change

    # Remove the trigger file
    rm /var/run/zypper-update-triggered
fi
```

#### Service Configuration (`/etc/systemd/system/zypper-offline-update.service`)

```ini
[Unit]
Description=Zypper Offline Update
DefaultDependencies=no
Conflicts=shutdown.target
After=network.target network-online.target
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/zypper-offline-update.sh

[Install]
WantedBy=multi-user.target
```

### 3. Zypper Update Notification (`zypper-update-notify-failure.service`)

This service checks if either the refresh and download service or the offline update service has failed, and sends a desktop notification if so.

#### Service Configuration (`/etc/systemd/system/zypper-update-notify-failure.service`)

```ini
[Unit]
Description=Notify if Zypper Update Services Failed
Wants=zypper-refresh-download.service zypper-offline-update.service
After=zypper-refresh-download.service zypper-offline-update.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c '\
    if systemctl --quiet is-failed zypper-refresh-download.service || \
       systemctl --quiet is-failed zypper-offline-update.service; then \
        /usr/bin/notify-send "Zypper Update Failed" "One of the Zypper services has failed. Please check the logs."; \
    fi'

[Install]
WantedBy=multi-user.target
```

## Installation and Usage

1. Save the scripts to their respective locations (`/usr/local/bin/`).
2. Make the scripts executable:
   ```
   sudo chmod +x /usr/local/bin/zypper-refresh-download.sh /usr/local/bin/zypper-offline-update.sh
   ```
3. Save the service files to `/etc/systemd/system/`.
4. Reload the systemd daemon:
   ```
   sudo systemctl daemon-reload
   ```
5. Enable and start the services:
   ```
   sudo systemctl enable --now zypper-refresh-download.service zypper-offline-update.service zypper-update-notify-failure.service
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
   journalctl -xe
   ```
3. Ensure that Zypper is working correctly by running basic commands manually:
   ```
   sudo zypper refresh
   sudo zypper dup --dry-run
   ```

## Contributing

Contributions to improve these services are welcome. Please submit pull requests or open issues on the project's GitHub repository.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
