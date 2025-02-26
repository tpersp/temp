# Raspberry Pi OS Lite Kiosk Mode Setup

This guide sets up **Raspberry Pi OS Lite (32-bit)** to automatically display a website in **Chromium kiosk mode** upon boot.

---

## **Important: Disable Wayland and Enable X11**

Before proceeding, ensure that Wayland is disabled and X11 is enabled.

To do this, open the Raspberry Pi configuration tool:

```sh
sudo raspi-config
```

Navigate to:
- **Advanced Options**
- **Wayland**
- Select **X11**
- Confirm with **Yes**

Then, restart your Raspberry Pi for the changes to take effect:

```sh
sudo reboot
```

---

## **1. Install Required Packages and Update System**

Update your system and install the necessary packages:

```sh
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium unclutter
```

- `xserver-xorg`: Provides the X11 server.
- `x11-xserver-utils`: Utility tools for X11.
- `xinit`: Allows starting an X session.
- `openbox`: A lightweight window manager.
- `chromium`: The Chromium browser.
- `unclutter`: Hides the mouse cursor when inactive.

---

## **2. Create an X11 Startup Script**

Create a script to launch the browser when X starts:

```sh
sudo nano /home/pi/start-browser.sh
```

Add the following:

```sh
#!/usr/bin/env bash
# Disable screen blanking
xset -dpms
xset s off
xset s noblank

# Launch openbox
openbox-session &

# Hide cursor after 1 second of inactivity
unclutter -idle 1 -root &

# Wait 5 seconds for Openbox to settle
sleep 5

# Launch Chromium in kiosk mode
chromium --force-x11 --noerrdialogs --disable-infobars --kiosk "http(s)://your-website-url"
```

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

Make it executable:

```sh
sudo chmod +x /home/pi/start-browser.sh
```

---

## **3. Create a systemd Service**

Create a systemd service to run the script at boot:

```sh
sudo nano /etc/systemd/system/kiosk.service
```

Add the following:

```ini
[Unit]
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
WantedBy=multi-user.target
```

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

---

## **4. Enable and Start the Service**

Run the following commands:

```sh
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service
sudo systemctl start kiosk.service
```

This will:
- Reload systemd to recognize the new service.
- Enable it to start on boot.
- Start it immediately.

---

## **5. Set Auto-Login and Reboot**

Ensure the Raspberry Pi auto-logs in and reboots:

```sh
sudo raspi-config nonint do_boot_behaviour B2
sudo reboot
```

After rebooting, your connected monitor should automatically display the specified website in **Chromium kiosk mode**.

---

## **Customization & Troubleshooting**

### **Change the Website**

To change the website displayed in kiosk mode, edit:

```sh
sudo nano /home/pi/start-browser.sh
```

Modify the URL in the `chromium` command and restart the service:

```sh
sudo systemctl restart kiosk.service
```

### **Change the scale/zoom**

To change the website scale/zoom in kiosk mode, edit:

```sh
sudo nano /home/pi/start-browser.sh
```

Add the `--force-device-scale-factor=1.5` flag before the URL.
This will add a 150% scale/zoom.
Remember to restart the service:

```sh
sudo systemctl restart kiosk.service
```



# Copy-Paste-Enter
**Note:** Remember to change the URL.
```sh
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium unclutter && \
echo -e '#!/usr/bin/env bash\n# Disable screen blanking\nxset -dpms\nxset s off\nxset s noblank\n\n# Launch openbox\nopenbox-session &\n\n# Hide cursor after 1 second of inactivity\nunclutter -idle 1 -root &\n\n# Wait 5 seconds for Openbox to settle\nsleep 5\n\n# Launch Chromium in kiosk mode\nchromium --force-x11 --noerrdialogs --disable-infobars --kiosk "https://www.erdetfredag.dk/"' | sudo tee /home/pi/start-browser.sh > /dev/null && \
sudo chmod +x /home/pi/start-browser.sh && \
echo -e '[Unit]\nDescription=Minimal X Kiosk\nAfter=systemd-user-sessions.service\nConflicts=getty@tty1.service\nAfter=getty@tty1.service\n\n[Service]\nUser=pi\nGroup=pi\nPAMName=login\nTTYPath=/dev/tty1\nTTYReset=yes\nTTYVHangup=yes\nType=simple\nStandardInput=tty\nStandardOutput=journal\nStandardError=journal\n\nExecStart=/usr/bin/xinit /home/pi/start-browser.sh -- :0 -nolisten tcp vt1\nRestart=always\nRestartSec=5\n\n[Install]\nWantedBy=multi-user.target' | sudo tee /etc/systemd/system/kiosk.service > /dev/null && \
sudo systemctl daemon-reload && \
sudo systemctl enable kiosk.service && \
sudo systemctl start kiosk.service && \
sudo raspi-config nonint do_boot_behaviour B2 && \
sudo reboot
```
