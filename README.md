# Raspberry Pi OS Lite Kiosk Mode Setup

This guide sets up **Raspberry Pi OS Lite (32-bit)** to automatically display a website in **Chromium kiosk mode** upon boot.

---

## **1. Install Required Packages and Update System**

We start by updating the system to ensure all installed packages are up to date and install only the necessary components to keep the system lightweight:

```sh
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium unclutter
```

### **Explanation of Installed Packages**
- `xserver-xorg`: Provides the X11 server, which is required to run graphical applications.
- `x11-xserver-utils`: Utility tools for X11, allowing additional configurations like display management.
- `xinit`: Allows starting an X session manually, which is necessary since we are not using a full desktop environment.
- `openbox`: A lightweight window manager that provides a minimal graphical environment.
- `chromium`: The Chromium browser, which will be used to display the website.
- `unclutter`: Hides the mouse cursor when inactive, keeping the kiosk display clean.

---

## **2. Create an X11 Startup Script**

We create a script to start the graphical session and launch the Chromium browser:

```sh
sudo nano /home/pi/start-browser.sh
```

Add the following:

```sh
#!/usr/bin/env bash
# Disable screen blanking to prevent the display from turning off
xset -dpms  # Disables Energy Star features
xset s off  # Turns off the screensaver
xset s noblank  # Prevents blank screen

# Launch Openbox, a minimal window manager
openbox-session &

# Hide the cursor after 1 second of inactivity
unclutter -idle 1 -root &

# Wait 5 seconds to allow Openbox to fully initialize
sleep 5

# Launch Chromium in kiosk mode, forcing it to use X11 and preventing pop-ups
chromium --force-x11 --noerrdialogs --disable-infobars --force-device-scale-factor=1.0 --kiosk "http(s)://your-website-url"
```

### **Why These Commands?**
- `xset` commands ensure the display remains on at all times.
- `openbox-session &` starts a minimal window manager to allow Chromium to run in full-screen.
- `unclutter` removes the cursor to make the display cleaner.
- `sleep 5` ensures that Openbox has fully loaded before launching Chromium.
- Chromium is launched in **kiosk mode**, which makes it run in fullscreen without user interface elements.

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

Make it executable:

```sh
sudo chmod +x /home/pi/start-browser.sh
```

---

## **3. Create a systemd Service**

We create a systemd service to automatically run the script on boot:

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

### **Why Use systemd?**
- Ensures the script starts automatically on boot.
- Runs under the `pi` user to avoid permission issues.
- If the script crashes, systemd will **restart it automatically** after 5 seconds.

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

---

## **4. Enable and Start the Service**

Run the following commands:

```sh
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service
sudo systemctl start kiosk.service
```

### **What These Commands Do:**
- `daemon-reload` reloads systemd to recognize the new service.
- `enable` ensures the service starts automatically on boot.
- `start` runs the service immediately.

---

## **5. Set Auto-Login and Reboot**

We need to configure the Raspberry Pi to automatically log in and run the script on startup:

```sh
sudo raspi-config nonint do_boot_behaviour B2
sudo reboot
```

### **Why This Step?**
- `do_boot_behaviour B2` sets the Pi to automatically log into the console session without requiring a manual login.
- `reboot` applies all changes and starts the kiosk automatically.

After rebooting, your monitor should display the specified website in **Chromium kiosk mode**.

---

## **Customization & Troubleshooting**

### **Change the Website**

To change the displayed website:

```sh
sudo nano /home/pi/start-browser.sh
```

Modify the URL in the `chromium` command and restart the service:

```sh
sudo systemctl restart kiosk.service
```

### **Change the Scale/Zoom**

To adjust the website scale, edit the `--force-device-scale-factor=X` flag in the Chromium command. This determines the zoom level of the displayed website.

#### **Examples:**
- `0.50`  = 50%   (More Zoomed Out)
- `0.75`  = 75%   (Zoomed Out)
- `1.0`   = 100%  (Default, No Scaling)
- `1.25`  = 125%  (Zoomed In)
- `1.5`   = 150%  (More Zoomed In)

Modify the command as needed for your setup, e.g.:

*Note: Do not use `**` around the number.*

```sh
chromium --force-x11 --noerrdialogs --disable-infobars --kiosk --force-device-scale-factor=**1.5** "http(s)://your-website-url"
```

Restart the service:

```sh
sudo systemctl restart kiosk.service
```

---

## **One-Line Installation Command**

To install everything in one step, use:
**NOTE:** Remember to change the URL.

```sh
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium unclutter && \
echo -e '#!/usr/bin/env bash\n# Disable screen blanking\nxset -dpms\nxset s off\nxset s noblank\n\n# Launch openbox\nopenbox-session &\n\n# Hide cursor after 1 second of inactivity\nunclutter -idle 1 -root &\n\n# Wait 5 seconds for Openbox to settle\nsleep 5\n\n# Launch Chromium in kiosk mode\nchromium --force-x11 --noerrdialogs --disable-infobars --force-device-scale-factor=1.0 --kiosk "https://www.erdetfredag.dk/"' | sudo tee /home/pi/start-browser.sh > /dev/null && \
sudo chmod +x /home/pi/start-browser.sh && \
echo -e '[Unit]\nDescription=Minimal X Kiosk\nAfter=systemd-user-sessions.service\nConflicts=getty@tty1.service\nAfter=getty@tty1.service\n\n[Service]\nUser=pi\nGroup=pi\nPAMName=login\nTTYPath=/dev/tty1\nTTYReset=yes\nTTYVHangup=yes\nType=simple\nStandardInput=tty\nStandardOutput=journal\nStandardError=journal\n\nExecStart=/usr/bin/xinit /home/pi/start-browser.sh -- :0 -nolisten tcp vt1\nRestart=always\nRestartSec=5\n\n[Install]\nWantedBy=multi-user.target' | sudo tee /etc/systemd/system/kiosk.service > /dev/null && \
sudo systemctl daemon-reload && \
sudo systemctl enable kiosk.service && \
sudo systemctl start kiosk.service && \
sudo raspi-config nonint do_boot_behaviour B2 && \
sudo reboot
```
