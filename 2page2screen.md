# Raspberry Pi OS Lite – Dual-Screen Chromium Kiosk Setup

This guide configures **Raspberry Pi OS Lite** to automatically launch Chromium in **dual-screen kiosk mode** using HDMI ports.

<!-- WHY: high-level overview -->
*We run a minimal X11 desktop (openbox) and open two Chromium windows—one per HDMI connector—each in kiosk mode.  
Scripts are wrapped in a systemd service so the Pi boots straight into the wallboard.*

It includes:

- Proper X11 session setup  
- Chromium installation with required dependencies  
- HDMI geometry detection for positioning windows  
- A login automation script using `xdotool`  
- Systemd service for autostart  
- Policy control to disable Chromium features like autofill and password manager  

---

## 0. Force Use of X11 (Disable Wayland)

<!-- WHY -->
Wayland breaks `xrandr` and Chromium’s `--force-x11` flag.  
Disabling Wayland guarantees the geometry checks will work.

```bash
sudo raspi-config
```

Navigate to:

```
Advanced Options → Wayland → Disable (Use X11)
```

Reboot:

```bash
sudo reboot
```

---

## 1. Update & Install Required Packages

<!-- WHY -->
Packages:

* X11 core: `xserver-xorg`, `x11-xserver-utils`, `xinit`  
* Window manager: `openbox`  
* Browser: `chromium` plus its shared-object deps  
* Kiosk helpers: `unclutter` (hide cursor), `xdotool` (scripted keys)  
* Misc: `upower` to silence power-save warnings  

```bash
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install -y \
    xserver-xorg \
    x11-xserver-utils \
    xinit \
    chromium \
    unclutter \
    xdotool \
    openbox \
    libgbm1 \
    libnss3 \
    libxss1 \
    libasound2 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libx11-xcb1 \
    fonts-liberation \
    upower
```

---

## 2. Configure HDMI Display Output (Force FKMS)

<!-- WHY -->
Fake-KMS keeps acceleration while staying X11-friendly.

```bash
sudo sed -i 's/^dtoverlay=vc4-kms-v3d/#dtoverlay=vc4-kms-v3d/' /boot/firmware/config.txt && \
echo -e 'dtoverlay=vc4-fkms-v3d\nhdmi_group=1\nhdmi_mode=16' | sudo tee -a /boot/firmware/config.txt > /dev/null
```

- `hdmi_group=1`, `hdmi_mode=16` = 1080p60. Change if you need another resolution.

---

## 3. Disable Chrome Autofill & Password Prompts

<!-- WHY -->
Disables any pop-ups that would appear on a public kiosk.

```bash
sudo mkdir -p /etc/opt/chrome/policies/managed && \
echo -e '{
  "AutoFillAddressEnabled": false,
  "AutoFillCreditCardEnabled": false,
  "PasswordManagerEnabled": false
}' | sudo tee /etc/opt/chrome/policies/managed/autofill_policy.json > /dev/null

sudo mkdir -p /etc/chromium/policies/managed && \
echo -e '{
  "AutoFillAddressEnabled": false,
  "AutoFillCreditCardEnabled": false,
  "PasswordManagerEnabled": false
}' | sudo tee /etc/chromium/policies/managed/autofill_policy.json > /dev/null
```

---

## 4. Automate Login with xdotool (Optional)

<!-- WHY -->
Types credentials once, then cookies keep the session alive.

Create the script:

```bash
sudo nano /home/pi/initial-login.sh
```

Paste:

```bash
#!/bin/bash
# Automate initial login using xdotool
sleep 30
xdotool search --sync --onlyvisible --class chromium windowactivate --sync
xdotool key Tab
sleep 2
xdotool type "itgsd"
sleep 2
xdotool key Tab
sleep 2
xdotool type "TempurSealy1234!"
sleep 2
xdotool key Return
sleep 15
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Right
sleep 2
xdotool key Right
sleep 2
xdotool key Right
sleep 2
xdotool key Right
sleep 2
xdotool key Down
sleep 2
xdotool key Down
sleep 2
xdotool key space
sleep 10
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool type "status IN (\"OPEN\") AND currentState != \"OK\""
sleep 2
xdotool key Return
```

```bash
sudo chmod +x /home/pi/initial-login.sh
```

---

## 5. Create the Kiosk Startup Script

<!-- WHY -->
Detects HDMI sizes, launches openbox, spawns two Chromium windows, then (optionally) the login macro.

```bash
sudo nano /home/pi/start-browser.sh
```

Paste full script, then:

```bash
sudo chmod +x /home/pi/start-browser.sh
```

---

## 6. Create a systemd Service

<!-- WHY -->
Boots straight into the kiosk and restarts on failure.

```bash
sudo nano /etc/systemd/system/kiosk.service
```

*(unit contents unchanged)*

Enable & start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable kiosk.service
sudo systemctl start kiosk.service
```

---

## 7. Enable Autologin and Reboot

```bash
sudo raspi-config nonint do_boot_behaviour B2
sudo reboot
```

---

## 8. Customisation—step-by-step edits

### 8.1 Change the displayed URLs  
1. Open the launcher:  
   ```bash
   sudo nano /home/pi/start-browser.sh
   ```  
2. Locate the two `--kiosk "http://…"` lines near the bottom.  
3. Replace each URL string with the new address.  
4. Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).  
5. Restart service:  
   ```bash
   sudo systemctl restart kiosk
   ```

---

### 8.2 Adjust page zoom / scale factor  
1. Edit `/home/pi/start-browser.sh`.  
2. Each Chromium command has a flag  
   `--force-device-scale-factor=VALUE`.  
   *Typical values:*  
   * 1.00 = 100 % (no zoom)  
   * 1.25 = 125 %  
   * 1.65 = 165 % (current left screen)  
3. Change the number, save, and restart the service.

---

### 8.3 Update username / password in the auto-login macro  
1. If you still use `/home/pi/initial-login.sh`, open it:  
   ```bash
   sudo nano /home/pi/initial-login.sh
   ```  
2. Overwrite the strings inside the `xdotool type "…"` lines.  
3. Keep the surrounding quotes.  
4. Save, exit, then restart the service.  
   ```bash
   sudo systemctl restart kiosk
   ```  
5. **To disable the macro entirely:**  
   ```bash
   sudo chmod -x /home/pi/initial-login.sh
   ```

---

## Final: Full Copy-Paste Install Script

   ```bash
sudo apt update && sudo apt -y full-upgrade && \
sudo apt install -y \
    xserver-xorg \
    x11-xserver-utils \
    xinit \
    chromium \
    unclutter \
    xdotool \
    libgbm1 \
    libnss3 \
    libxss1 \
    libasound2 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libx11-xcb1 \
    fonts-liberation \
    upower && \
sudo sed -i 's/^dtoverlay=vc4-kms-v3d/#dtoverlay=vc4-kms-v3d/' /boot/firmware/config.txt && \
echo -e 'dtoverlay=vc4-fkms-v3d\nhdmi_group=1\nhdmi_mode=16' | sudo tee -a /boot/firmware/config.txt > /dev/null && \
sudo mkdir -p /etc/opt/chrome/policies/managed && \
echo -e '{
    "AutoFillAddressEnabled": false,
    "AutoFillCreditCardEnabled": false,
    "PasswordManagerEnabled": false
}' | sudo tee /etc/opt/chrome/policies/managed/autofill_policy.json > /dev/null && \
sudo mkdir -p /etc/chromium/policies/managed && \
echo -e '{
    "AutoFillAddressEnabled": false,
    "AutoFillCreditCardEnabled": false,
    "PasswordManagerEnabled": false
}' | sudo tee /etc/chromium/policies/managed/autofill_policy.json > /dev/null && \
echo -e '#!/bin/bash
# Automate initial login using xdotool
sleep 30
xdotool search --sync --onlyvisible --class chromium windowactivate --sync
xdotool key Tab
sleep 2
xdotool type "itgsd"
sleep 2
xdotool key Tab
sleep 2
xdotool type "TempurSealy1234!"
sleep 2
xdotool key Return
sleep 15
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Right
sleep 2
xdotool key Right
sleep 2
xdotool key Right
sleep 2
xdotool key Right
sleep 2
xdotool key Down
sleep 2
xdotool key Down
sleep 2
xdotool key space
sleep 10
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool key Tab
sleep 2
xdotool type "status IN (\"OPEN\") AND currentState != \"OK\""
sleep 2
xdotool key Return
' | sudo tee /home/pi/initial-login.sh > /dev/null && \
sudo chmod +x /home/pi/initial-login.sh && \
echo -e '#!/bin/bash
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
chromium --force-x11 \
        --disable-gpu --no-sandbox --noerrdialogs --disable-infobars \
		--disable-extensions \
        --force-device-scale-factor=1.65 \
        --user-data-dir=/tmp/chromium-hdmi1 \
        --window-position=${posX1},${posY1} --window-size=${width1},${height1} \
        --kiosk "http://wallboard.tempur.com:3000/devbuild" &

# Short delay to avoid any race conditions.
sleep 2

# Launch Chromium for HDMI-2 with its own user-data directory.
chromium --force-x11 \
        --disable-gpu --no-sandbox --noerrdialogs --disable-infobars \
		--disable-extensions \
        --force-device-scale-factor=1.0 \
        --user-data-dir=/tmp/chromium-hdmi2 \
        --window-position=${posX2},${posY2} --window-size=${width2},${height2} \
        --kiosk "https://danofficeit.app.opsramp.com/portal/infra-ui/alerts" &

# Short delay to ensure the browser is fully loaded.
sleep 20

# Run the initial login script every time the service starts
/home/pi/initial-login.sh

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

Environment=DISPLAY=:0
Environment=XAUTHORITY=/home/pi/.Xauthority

ExecStart=/usr/bin/xinit /home/pi/start-browser.sh -- :0 -nolisten tcp vt1
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target' | sudo tee /etc/systemd/system/kiosk.service > /dev/null && \
sudo systemctl daemon-reload && \
sudo systemctl enable kiosk.service && \
sudo systemctl start kiosk.service && \
sudo raspi-config nonint do_boot_behaviour B2 && \


//Note: Write down detailed description of each part of the code, so a successor to me can read and understand it all, make changes where needed and easily maintain the code.
sudo reboot
   ```  
