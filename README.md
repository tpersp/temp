# Raspberry Pi OS Lite Kiosk Mode Setup

This repository contains guides for setting up Raspberry Pi OS Lite to automatically display websites using **Chromium in kiosk mode** upon boot. It includes instructions for single-screen and dual-screen setups, ensuring a lightweight, efficient, and automated kiosk solution ideal for dashboards, digital signage, or information displays.

## Contents

- **[Single-Screen Kiosk Mode](1page1screen.md)**: Step-by-step guide for configuring a Raspberry Pi OS Lite device to automatically boot and display a single webpage in Chromium browser kiosk mode.

- **[Dual-Screen Kiosk Mode](2page2screen.md)**: Detailed instructions for setting up a Raspberry Pi to simultaneously display two different websites on two separate HDMI-connected screens.

## Features

- **Minimalistic Setup**: Utilizes lightweight packages like Xorg, Openbox, and Chromium.
- **Automated Boot**: Leveraging `systemd` to ensure automatic startup of kiosk mode upon boot.
- **Customization**: Easily configurable to change URLs, adjust zoom levels, and manage display settings.
- **Cursor Management**: Incorporates `unclutter` to hide the cursor, ensuring a clean and professional presentation.

## Quick Installation

Each guide provides a convenient one-line installation command to quickly deploy the kiosk mode, minimizing manual setup efforts.

## Use Cases

- Digital Signage
- Informational Displays
- Monitoring Dashboards
- Exhibitions and Conferences

For detailed setup instructions and customization options, refer to the respective markdown files included in this repository.

