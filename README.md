# Raspberry Pi OS Lite Kiosk Mode Setup

This guide will configure a **Raspberry Pi OS Lite (32-bit)** installation to display a website automatically on a connected monitor upon boot using **Chromium in kiosk mode**.

## **1. Install Required Packages**
First, install the necessary software:

```sh
sudo apt update
sudo apt install --no-install-recommends xserver-xorg x11-xserver-utils xinit openbox chromium-browser
```

- `xserver-xorg`: Provides the X11 server.
- `x11-xserver-utils`: Utility tools for X11.
- `xinit`: Allows starting an X session.
- `openbox`: A lightweight window manager.
- `chromium-browser`: The Chromium browser.

---

## **2. Create an X11 Startup Script**
We need to create a script that will launch the browser when X starts.

```sh
nano /home/pi/start-browser.sh
```

Add the following:

```sh
#!/bin/bash
xset -dpms          # Disable display power management
xset s off          # Disable screen saver
xset s noblank      # Prevent screen from blanking

# Start Openbox window manager
openbox-session &

# Wait a bit for Openbox to initialize
sleep 2

# Launch Chromium in kiosk mode
chromium-browser --noerrdialogs --disable-infobars --kiosk 'http://your-website-url'
```

Replace `'http://your-website-url'` with the actual URL you want to display.

Save the file (`CTRL+X`, then `Y`, then `Enter`).

Make it executable:

```sh
chmod +x /home/pi/start-browser.sh
```

---

## **3. Create a systemd Service**
Now, create a systemd service to run this script on boot.

```sh
sudo nano /etc/systemd/system/kiosk.service
```

Add the following:

```ini
[Unit]
Description=Chromium Kiosk Mode
After=multi-user.target

[Service]
User=pi
Environment=DISPLAY=:0
ExecStart=/home/pi/start-browser.sh
Restart=always
RestartSec=5
StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

---

## **4. Enable and Start the Service**
Run:

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

## **5. Test and Reboot**
Reboot your Raspberry Pi to verify everything works:

```sh
sudo reboot
```

After boot, your connected monitor should automatically launch the Chromium browser in **kiosk mode** displaying the specified website.

---

## **Optional: Auto-Login for pi User**
If your Pi doesn’t automatically log in as `pi`, enable auto-login:

```sh
sudo raspi-config
```
- Go to **System Options** → **Boot/Auto Login** → **Console Autologin**.

Reboot, and it should work as expected.

---

## **Customization**
- To exit kiosk mode, press `CTRL + ALT + F1`, log in, and disable the service with:
  ```sh
  sudo systemctl stop kiosk.service
  ```
- To change the website, edit `/home/pi/start-browser.sh` and restart the service.

---

This setup ensures a lightweight, stable, and reliable way to show a website on boot using Raspberry Pi OS Lite.

# One-Liner
**Note:** Remember to change the URL.
```sh
sudo apt update && sudo apt install --no-install-recommends -y xserver-xorg x11-xserver-utils xinit openbox chromium-browser && echo -e '#!/bin/bash\nxset -dpms\nxset s off\nxset s noblank\nopenbox-session &\nsleep 2\nchromium-browser --noerrdialogs --disable-infobars --kiosk "http://your-website-url"' | sudo tee /home/pi/start-browser.sh > /dev/null && sudo chmod +x /home/pi/start-browser.sh && echo -e '[Unit]\nDescription=Chromium Kiosk Mode\nAfter=multi-user.target\n\n[Service]\nUser=pi\nEnvironment=DISPLAY=:0\nExecStart=/home/pi/start-browser.sh\nRestart=always\nRestartSec=5\nStandardOutput=syslog\nStandardError=syslog\n\n[Install]\nWantedBy=multi-user.target' | sudo tee /etc/systemd/system/kiosk.service > /dev/null && sudo systemctl daemon-reload && sudo systemctl enable kiosk.service && sudo systemctl start kiosk.service && sudo raspi-config nonint do_boot_behaviour B2 && sudo reboot
```
