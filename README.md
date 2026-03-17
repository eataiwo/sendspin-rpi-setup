# Sendspin CLI — Raspberry Pi Bare Metal Setup Guide

A step-by-step guide to installing `sendspin-cli` as a persistent, auto-starting background service on Raspberry Pi OS (Bookworm). This turns any Pi with an audio output into a synchronised multi-room audio receiver for a [Music Assistant](https://www.music-assistant.io/) server running Sendspin.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Update the System](#1-update-the-system)
- [2. Install uv (Python Package Runner)](#2-install-uv-python-package-runner)
- [3. Test sendspin-cli](#3-test-sendspin-cli)
- [4. Identify Your Audio Output Device](#4-identify-your-audio-output-device)
- [5. Install sendspin as a Persistent Tool](#5-install-sendspin-as-a-persistent-tool)
- [6. Create the systemd Service](#6-create-the-systemd-service)
- [7. Configure the Service](#7-configure-the-service)
- [8. Enable and Start the Service](#8-enable-and-start-the-service)
- [9. Verify It's Working](#9-verify-its-working)
- [10. Updating sendspin](#10-updating-sendspin)
- [Troubleshooting](#troubleshooting)
- [Reference: Useful Commands](#reference-useful-commands)

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Raspberry Pi 3B+ / 4 / 5 / Zero 2W | Pi Zero 2W is fine as a *client* only |
| Raspberry Pi OS **Bookworm** (64-bit recommended) | Avoid Trixie for now — known issues |
| Python 3.12+ | Comes with Bookworm |
| Audio output | 3.5mm jack, USB DAC, or DAC HAT |
| Network | Same LAN/subnet as your Music Assistant server |
| Music Assistant server | Running Sendspin (MA 2.7+) elsewhere on the network |

> **Note:** This Pi is a Sendspin *client/receiver* only. The Music Assistant server should run on a separate, more powerful machine (Pi 4/5, NAS, etc).

---

## 1. Update the System

Always start with a fully updated system to avoid dependency issues.

```bash
sudo apt update && sudo apt upgrade -y
```

Reboot if a kernel update was applied:

```bash
sudo reboot
```

---

## 2. Install uv (Python Package Runner)

`uv` is a fast Python package manager used to run and install `sendspin` without polluting the system Python. It's the official recommended way to install sendspin.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Add `uv` to your current shell session's PATH:

```bash
source $HOME/.local/bin/env
```

Verify the install:

```bash
uv --version
```

---

## 3. Test sendspin-cli

Before setting up the service, do a quick interactive test to confirm everything works. This will connect to any Sendspin server it finds on the network via mDNS.

```bash
uvx sendspin
```

You should see a terminal UI appear. If your Music Assistant server is running, this Pi should appear as a new player in the MA interface within a few seconds.

Press `q` to quit when done.

> **Troubleshooting:** If it connects but there's no sound, check [audio device selection](#4-identify-your-audio-output-device) below.

---

## 4. Identify Your Audio Output Device

### List available audio devices

```bash
uvx sendspin --list-devices
```

This outputs something like:

```
Available audio output devices:
  [0] default - Default Audio Device (2ch, 48000 Hz)
  [1] hw:0,0 - bcm2835 Headphones (2ch, 48000 Hz)       ← 3.5mm jack
  [2] hw:1,0 - USB Audio Device (2ch, 48000 Hz)          ← USB DAC
  [3] hw:2,0 - snd_rpi_hifiberry_dacplus (2ch, 192000 Hz) ← DAC HAT
```

Note the device ID or name you want to use. For most setups:

- **3.5mm headphone jack:** usually `hw:0,0` or the `bcm2835` entry
- **USB DAC:** usually `hw:1,0` or named `USB Audio`
- **HiFiBerry / other DAC HAT:** named accordingly

### Test audio output on that device

Replace `hw:1,0` with your device:

```bash
uvx sendspin --device hw:1,0
```

Play something from Music Assistant — confirm you hear audio from the right output.

---

## 5. Install sendspin as a Persistent Tool

Rather than using `uvx` (which runs from cache), install sendspin as a proper tool so the binary is available system-wide for the service:

```bash
uv tool install sendspin
```

Verify it installed:

```bash
sendspin --version
```

> If `sendspin` isn't found, the uv tools bin directory may not be on your PATH. Add it:
> ```bash
> echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
> source ~/.bashrc
> ```

---

## 6. Create the systemd Service

Create the service file. Replace the placeholder values in the `[Service]` section — see [Step 7](#7-configure-the-service) for all options.

```bash
sudo nano /etc/systemd/system/sendspin.service
```

Paste the following (adjust the values marked with `# ← CHANGE THIS`):

```ini
[Unit]
Description=Sendspin Audio Client
Documentation=https://github.com/Sendspin/sendspin-cli
After=network-online.target sound.target
Wants=network-online.target

[Service]
Type=simple
User=pi                                         # ← CHANGE THIS to your username
Environment="PATH=/home/pi/.local/bin:/usr/bin:/bin"  # ← CHANGE pi to your username
ExecStart=/home/pi/.local/bin/sendspin daemon \  # ← CHANGE pi to your username
    --name "Living Room" \                       # ← CHANGE to your room name
    --device hw:1,0 \                            # ← CHANGE to your audio device
    --hardware-volume true
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

---

## 7. Configure the Service

Key options you can add to the `ExecStart` line:

| Option | Example | Description |
|---|---|---|
| `--name` | `--name "Kitchen"` | Friendly name shown in Music Assistant |
| `--device` | `--device hw:1,0` | Audio output device to use |
| `--hardware-volume` | `--hardware-volume true` | Let Sendspin control system volume |
| `--delay` | `--delay -100` | Adjust sync in milliseconds (negative = earlier) |
| `--url` | `--url ws://192.168.1.50:8097` | Pin to a specific MA server instead of mDNS auto-discovery |
| `--id` | `--id kitchen-pi` | Stable unique ID (persists group membership across reboots) |
| `--format` | `--format flac` | Preferred audio format (`flac`, `opus`, `pcm`) |

### Recommended additions

**Give the player a stable ID** so Music Assistant remembers its group settings after restarts:

```ini
ExecStart=/home/pi/.local/bin/sendspin daemon \
    --name "Kitchen" \
    --id kitchen-rpi \
    --device hw:1,0 \
    --hardware-volume true
```

**Pin to a specific server** if mDNS is unreliable on your network:

```ini
ExecStart=/home/pi/.local/bin/sendspin daemon \
    --name "Kitchen" \
    --id kitchen-rpi \
    --device hw:1,0 \
    --url ws://192.168.1.50:8097
```

After any changes to the service file, reload the daemon:

```bash
sudo systemctl daemon-reload
```

---

## 8. Enable and Start the Service

```bash
# Enable the service to start on every boot
sudo systemctl enable sendspin

# Start it now without needing a reboot
sudo systemctl start sendspin
```

---

## 9. Verify It's Working

### Check the service status

```bash
sudo systemctl status sendspin
```

You should see `Active: active (running)`.

### Watch live logs

```bash
journalctl -u sendspin -f
```

You should see output like:

```
sendspin[1234]: Connecting to Sendspin server at ws://192.168.1.50:8097
sendspin[1234]: Connected as "Kitchen" (id: kitchen-rpi)
sendspin[1234]: State: synchronized
```

### Confirm it appears in Music Assistant

Open your Music Assistant web UI → **Players**. Your Pi should appear as a new player with the name you set. You can now add it to a sync group and play music to it.

### Test a reboot

```bash
sudo reboot
```

After it comes back up, check the service started automatically:

```bash
sudo systemctl status sendspin
```

---

## 10. Updating sendspin

```bash
# Update to the latest version
uv tool upgrade sendspin

# Restart the service to use the new version
sudo systemctl restart sendspin

# Verify the new version
sendspin --version
```

---

## Troubleshooting

### Service fails to start

Check logs for the actual error:

```bash
journalctl -u sendspin -n 50 --no-pager
```

Common causes:
- Wrong path to the `sendspin` binary — verify with `which sendspin`
- Wrong username in `User=` or `Environment=` lines
- Audio device not available at boot time (add `After=sound.target`)

### No sound / wrong audio device

List devices again and re-test interactively:

```bash
uvx sendspin --list-devices
uvx sendspin --device hw:2,0   # test a different device
```

Then update the `--device` value in your service file and reload:

```bash
sudo systemctl daemon-reload && sudo systemctl restart sendspin
```

### Player doesn't appear in Music Assistant

- Confirm both devices are on the **same network/subnet**
- Check mDNS is not being blocked (Pi-hole, AdGuard, VLANs can all cause this)
- Try pinning with `--url ws://<ma-server-ip>:8097` instead of mDNS discovery
- Ensure MA server is running version 2.7 or later (Sendspin support)

### Audio is out of sync with other rooms

Adjust the delay on the affected Pi:

```ini
ExecStart=/home/pi/.local/bin/sendspin daemon \
    --name "Kitchen" \
    --device hw:1,0 \
    --delay -150        # try values like -50, -100, -150, -200
```

A negative delay compensates for hardware audio buffering. Reload and restart after each change, then test with a group play.

### mDNS discovery is unreliable

Some routers and managed switches block mDNS multicast. Pin the client directly to the server's IP:

```ini
--url ws://192.168.1.50:8097
```

---

## Reference: Useful Commands

```bash
# Start / stop / restart the service
sudo systemctl start sendspin
sudo systemctl stop sendspin
sudo systemctl restart sendspin

# Check status
sudo systemctl status sendspin

# View logs (live)
journalctl -u sendspin -f

# View last 100 log lines
journalctl -u sendspin -n 100 --no-pager

# Disable autostart (without stopping it now)
sudo systemctl disable sendspin

# Re-enable autostart
sudo systemctl enable sendspin

# Edit the service file
sudo nano /etc/systemd/system/sendspin.service
# After editing, always reload:
sudo systemctl daemon-reload

# Update sendspin
uv tool upgrade sendspin && sudo systemctl restart sendspin

# Test interactively (useful for debugging)
uvx sendspin
uvx sendspin --list-devices
uvx sendspin --device hw:1,0 --name "Test"
```

---

## Notes
- The Sendspin protocol and `sendspin-cli` are currently in **technical preview** — the API may change between versions. Pin to a specific version with `uv tool install sendspin==x.y.z` if you need stability.
- Official Sendspin documentation: [sendspin-audio.com](https://www.sendspin-audio.com/)
- sendspin-cli source: [github.com/Sendspin/sendspin-cli](https://github.com/Sendspin/sendspin-cli)
