# KDE Cursor Hiding Script

This script automatically hides the mouse cursor in KDE after a period of inactivity.

## Description

The script waits for the KDE window manager (KWin) to start, then launches the `unclutter` tool to hide the mouse cursor after 2 seconds of inactivity. It's designed to run at KDE startup.

## Prerequisites

- KDE Plasma desktop environment
- `unclutter` tool installed
- `pgrep` command (usually pre-installed on most Linux distributions)

## Installation

1. Save the script to a suitable location, e.g., `~/.local/bin/hide_cursor.sh`

2. Make the script executable:
   ```
   chmod +x ~/.local/bin/hide_cursor.sh
   ```

3. To run at KDE startup, create a file `~/.config/autostart/hide_cursor.desktop` with the following content:
   ```
   [Desktop Entry]
   Type=Application
   Name=Hide Cursor
   Exec=/home/yourusername/.local/bin/hide_cursor.sh
   X-GNOME-Autostart-enabled=true
   ```
   Replace "yourusername" with your actual username.

## Usage

The script will run automatically at KDE startup if set up as described above. To run manually:

```
~/.local/bin/hide_cursor.sh
```

## Customization

- To change the inactivity period, modify the `-idle 2` part of the `unclutter` command in the script. The number represents seconds.

## Troubleshooting

If the cursor doesn't hide:
1. Ensure `unclutter` is installed
2. Check if the script is running: `pgrep -f hide_cursor.sh`
3. Look for error messages: `journalctl -xe | grep hide_cursor.sh`

## License

This script is provided "as is", without warranty of any kind. You are free to modify and distribute it.
