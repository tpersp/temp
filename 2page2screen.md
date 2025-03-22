# Raspberry Pi OS Lite Dual-Screen Kiosk Mode Setup

This guide sets up **Raspberry Pi OS Lite** to automatically display websites on two separate HDMI-connected screens using **Chromium in kiosk mode** upon boot.

---

## **0. Configure Raspberry Pi to Use X11**

Ensure your Raspberry Pi is configured to use X11 instead of Wayland:

```sh
sudo raspi-config
```

Navigate to:

```
Advanced Options → Wayland →  Use X11 (Disable Wayland)
```

Reboot the Raspberry Pi after applying this change.

---

## **1. Update System and Install Required Packages**

Update the system and install minimal necessary components:

```sh
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium unclutter
```

### **Installed Packages Explained**
- `xserver-xorg`: X11 server for graphical applications.
- `x11-xserver-utils`: X11 utilities for display management.
- `xinit`: Starts X sessions without a full desktop.
- `openbox`: Minimal window manager.
- `chromium`: Browser for website display.
- `unclutter`: Hides mouse cursor when inactive.

---

## **2. Create an X11 Startup Script**

Create a script to initialize displays and launch Chromium on both HDMI outputs:

```sh
sudo nano /home/pi/start-browser.sh
```

Add this script:

```sh
#!/usr/bin/env bash
# Ensure X11 session
if [ "$XDG_SESSION_TYPE" = "wayland" ] || [ -n "$WAYLAND_DISPLAY" ]; then
    echo "Error: Wayland session detected. X11 is required. Exiting."
    exit 1
fi

# Disable screen blanking
xset -dpms
xset s off
xset s noblank

sleep 2
xrandr --auto
sleep 2

# Detect HDMI geometries
HDMI1_geom=$(xrandr --query | grep -E "^(HDMI|HDMI-A)[-]?1 connected" | sed -n "s/.* connected.* \([0-9]\+x[0-9]\++[0-9]\++[0-9]\+\).*/\1/p")
HDMI2_geom=$(xrandr --query | grep -E "^(HDMI|HDMI-A)[-]?2 connected" | sed -n "s/.* connected.* \([0-9]\+x[0-9]\++[0-9]\++[0-9]\+\).*/\1/p")

if [ -z "$HDMI1_geom" ] || [ -z "$HDMI2_geom" ]; then
    echo "Error: Unable to detect both HDMI screens. Exiting."
    exit 1
fi

IFS="x+"
read width1 height1 posX1 posY1 <<< "$HDMI1_geom"
read width2 height2 posX2 posY2 <<< "$HDMI2_geom"

openbox-session &
unclutter -idle 1 -root &
sleep 5

# Chromium for HDMI-1
chromium --force-x11 --noerrdialogs --disable-infobars --force-device-scale-factor=1.0 \
         --user-data-dir=/tmp/chromium-hdmi1 \
         --window-position=${posX1},${posY1} --window-size=${width1},${height1} \
         --kiosk "https://example.com/" &
sleep 2

# Chromium for HDMI-2
chromium --force-x11 --noerrdialogs --disable-infobars --force-device-scale-factor=1.0 \
         --user-data-dir=/tmp/chromium-hdmi2 \
         --window-position=${posX2},${posY2} --window-size=${width2},${height2} \
         --kiosk "https://example1.com/" &

# Keep session alive
while true; do
    sleep 60
done
```

Save and exit (`CTRL+X`, then `Y`, then `Enter`). Make it executable:

```sh
sudo chmod +x /home/pi/start-browser.sh
```

---

## **3. Create systemd Service**

Create a systemd service to auto-start the kiosk:

```sh
sudo nano /etc/systemd/system/kiosk.service
```

Add the following:

```ini
[Unit]
Description=Dual HDMI X Kiosk
After=systemd-user-sessions.service
Conflicts=getty@tty1.service
After=getty@tty1.service

[Service]
User=pi
Group=pi
PAMName=login
TTYPath=/dev/tty1
TTYReset=yes
TTYVHangup=yes
Type=simple
StandardInput=tty
StandardOutput=journal
StandardError=journal

ExecStart=/usr/bin/xinit /home/pi/start-browser.sh -- :0 -nolisten tcp vt1
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

---

## **4. Enable and Start the Service**

```sh
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service
sudo systemctl start kiosk.service
```

- `daemon-reload`: Reloads systemd configuration.
- `enable`: Enables service to start at boot.
- `start`: Starts service immediately.

---

## **5. Set Auto-Login and Reboot**

Configure automatic login:

```sh
sudo raspi-config nonint do_boot_behaviour B2
sudo reboot
```

This sets automatic login and starts kiosk mode on reboot.

---

## **Customization & Troubleshooting**

### **Change Displayed Websites**

Edit URLs in `/home/pi/start-browser.sh` and restart:

```sh
sudo systemctl restart kiosk.service
```

### **Adjust Scale or Zoom**

Edit `--force-device-scale-factor=X`:
- `0.75` (Zoom Out)
- `1.0` (Default)
- `1.25` (Zoom In)

Restart the service after changes.

---

## **One-Line Installation Command**

Run all setup steps at once (change URLs accordingly):

```sh
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium unclutter && \
echo -e '
#!/usr/bin/env bash
# Check if the session is running under Wayland
if [ "$XDG_SESSION_TYPE" = "wayland" ] || [ -n "$WAYLAND_DISPLAY" ]; then
    echo "Error: Detected Wayland session. This kiosk requires an X11 session. Exiting."
    exit 1
fi

# Disable screen blanking
xset -dpms
xset s off
xset s noblank

sleep 2
xrandr --auto
sleep 2

# Debug: output current xrandr info
echo "xrandr --query output:"
xrandr --query

# Get geometry for HDMI-1 and HDMI-2 using a flexible regex.
HDMI1_geom=$(xrandr --query | grep -E "^(HDMI|HDMI-A)[-]?1 connected" | sed -n "s/.* connected.* \([0-9]\+x[0-9]\++[0-9]\++[0-9]\+\).*/\1/p")
HDMI2_geom=$(xrandr --query | grep -E "^(HDMI|HDMI-A)[-]?2 connected" | sed -n "s/.* connected.* \([0-9]\+x[0-9]\++[0-9]\++[0-9]\+\).*/\1/p")

echo "Detected HDMI-1 geometry: $HDMI1_geom"
echo "Detected HDMI-2 geometry: $HDMI2_geom"

if [ -z "$HDMI1_geom" ]; then
    echo "Error: HDMI-1 not found or unable to determine geometry."
    exit 1
fi
if [ -z "$HDMI2_geom" ]; then
    echo "Error: HDMI-2 not found or unable to determine geometry."
    exit 1
fi

IFS="x+"
read width1 height1 posX1 posY1 <<< "$HDMI1_geom"
read width2 height2 posX2 posY2 <<< "$HDMI2_geom"

echo "HDMI-1: width=$width1, height=$height1, posX=$posX1, posY=$posY1"
echo "HDMI-2: width=$width2, height=$height2, posX=$posX2, posY=$posY2"

openbox-session &
unclutter -idle 1 -root &
sleep 5

# Launch Chromium for HDMI-1 with a dedicated user-data directory.
chromium --force-x11 --noerrdialogs --disable-infobars --force-device-scale-factor=1.0 \
         --user-data-dir=/tmp/chromium-hdmi1 \
         --window-position=${posX1},${posY1} --window-size=${width1},${height1} \
         --kiosk "https://example.com/" &

# Short delay to avoid any race conditions.
sleep 2

# Launch Chromium for HDMI-2 with its own user-data directory.
chromium --force-x11 --noerrdialogs --disable-infobars --force-device-scale-factor=1.0 \
         --user-data-dir=/tmp/chromium-hdmi2 \
         --window-position=${posX2},${posY2} --window-size=${width2},${height2} \
         --kiosk "https://example1.com/" &

# Keep the session alive indefinitely.
while true; do
    sleep 60
done
' | sudo tee /home/pi/start-browser.sh > /dev/null && \
sudo chmod +x /home/pi/start-browser.sh && \
echo -e '[Unit]
Description=Minimal X Kiosk
After=systemd-user-sessions.service
Conflicts=getty@tty1.service
After=getty@tty1.service

[Service]
User=pi
Group=pi
PAMName=login
TTYPath=/dev/tty1
TTYReset=yes
TTYVHangup=yes
Type=simple
StandardInput=tty
StandardOutput=journal
StandardError=journal

ExecStart=/usr/bin/xinit /home/pi/start-browser.sh -- :0 -nolisten tcp vt1
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target' | sudo tee /etc/systemd/system/kiosk.service > /dev/null && \
sudo systemctl daemon-reload && \
sudo systemctl enable kiosk.service && \
sudo systemctl start kiosk.service && \
sudo raspi-config nonint do_boot_behaviour B2 && \
sudo reboot
```

Reboot to finalize the setup.

