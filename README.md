### Pi Setup (This was specifically built for the Pi 4B with Seeed 2 mic array but should work with most others - Not tested on Pi 5 which may different issues)
Raspberry Pi Imager
- Raspberry Pi OS Lite (64-bit)

-----------------------

# Auto install - Clone this repo to your Pi and

```bash
git clone https://github.com/dihan/pi-voice-assistant
chmod +x install.sh
sudo ./install.sh
```

-----------------------

# Manual Install (Recommended - Easier to debug)

## Raspberry Pi Voice Assistant Setup Guide

A comprehensive guide for setting up a voice assistant system using Wyoming Satellite and OpenWakeword on a Raspberry Pi with ReSpeaker microphone support.

## Prerequisites

- Raspberry Pi with Debian/Ubuntu-based OS
- ReSpeaker microphone array
- Speaker setup
- Internet connection
- Basic knowledge of Linux commands

## Initial SSH Setup

Generate a new SSH key:
```bash
ssh-keygen -t ed25519 -C "your.email@example.com"

# Start the SSH agent
eval "$(ssh-agent -s)"

# Add your SSH key to the agent
ssh-add ~/.ssh/id_ed25519

# Display your public key
cat ~/.ssh/id_ed25519.pub
```

### Reinstalling SSH with New Key

1. Remove old host key:
```bash
ssh-keygen -R raspberrypi4.local
```

2. Copy your public key to the Raspberry Pi:
```bash
ssh-copy-id username@raspberrypi4.local
```


-----------------------


## Installation Steps

### 1. ReSpeaker Setup (v6.6) - (Do this first as drivers are a mess to get it to work and need to use a fork) 

```bash
sudo apt install git
git clone https://github.com/HinTak/seeed-voicecard
cd seeed-voicecard
git checkout v6.6
sudo ./install.sh
sudo reboot
```

Test the microphone setup:
```bash
# Record test
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav

# Playback test
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav

# Access mixer settings
alsamixer -c seeed2micvoicec
```

### 2. Install Required Packages

```bash
sudo apt-get update
sudo apt-get install --no-install-recommends git python3-venv libopenblas-dev
```

### 3. Wyoming Satellite Setup

```bash
# Clone and setup Wyoming Satellite
git clone https://github.com/rhasspy/wyoming-satellite.git
cd wyoming-satellite/

# Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip3 install --upgrade pip wheel setuptools
pip3 install -f 'https://synesthesiam.github.io/prebuilt-apps/' \
    -r requirements.txt \
    -r requirements_audio_enhancement.txt \
    -r requirements_vad.txt

pip install .
```

### 4. OpenWakeword Setup

```bash
cd ..
git clone https://github.com/rhasspy/wyoming-openwakeword.git
cd wyoming-openwakeword
script/setup
```

### 5. Service Configuration

#### OpenWakeword Service

Create `/etc/systemd/system/wyoming-openwakeword.service`:
```bash
sudo nano /etc/systemd/system/wyoming-openwakeword.service
```

```ini
[Unit]
Description=Wyoming openWakeWord

[Service]
Type=simple
ExecStart=/home/username/wyoming-openwakeword/script/run --uri 'tcp://0.0.0.0:10400'
WorkingDirectory=/home/username/wyoming-openwakeword
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

#### Wyoming Satellite Service

Create `/etc/systemd/system/wyoming-satellite.service`:
```bash
sudo nano /etc/systemd/system/wyoming-satellite.service
```

```ini
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target
Requires=wyoming-openwakeword.service

[Service]
Type=simple
ExecStart=/home/username/wyoming-satellite/script/run \
  --debug \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw'
WorkingDirectory=/home/username/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

### 6. LED Service Setup

```bash
# Setup LED service environment
cd wyoming-satellite/examples
python3 -m venv --system-site-packages .venv
.venv/bin/pip3 install --upgrade pip wheel setuptools
.venv/bin/pip3 install 'wyoming==1.5.2'

# Install required system packages
sudo apt-get install python3-spidev python3-gpiozero
```

Create `/etc/systemd/system/2mic_leds.service`:
```bash
sudo nano /etc/systemd/system/wyoming-satellite.service
```

```ini
[Unit]
Description=2Mic LEDs

[Service]
Type=simple
ExecStart=/home/username/wyoming-satellite/examples/.venv/bin/python3 2mic_service.py --uri 'tcp://127.0.0.1:10500' --led-brightness 10
WorkingDirectory=/home/username/wyoming-satellite/examples
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

<!-- sudo nano /home/username/wyoming-satellite/examples/2mic_service.py -->
### 7. Enable and Start Services

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable services
sudo systemctl enable wyoming-satellite.service wyoming-openwakeword.service 2mic_leds.service

# Start services
sudo systemctl start wyoming-satellite.service wyoming-openwakeword.service 2mic_leds.service
```

## Debugging and Maintenance

### Service Management

```bash
# Check service status
sudo systemctl status wyoming-satellite
sudo systemctl status wyoming-openwakeword
sudo systemctl status 2mic_leds

# View logs
journalctl -u wyoming-satellite.service -f
journalctl -u wyoming-openwakeword.service -f
journalctl -u 2mic_leds.service -f
```

### Audio Testing

```bash
# List recording devices
arecord -L

# List playback devices
aplay -L

# Test recording
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav

# Test playback
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav
```

## Troubleshooting Tips

1. **Audio Issues**
   - Check device permissions
   - Add user to audio group: `sudo usermod -aG audio $(whoami)`
   - Verify device names with `arecord -L` and `aplay -L`

2. **Service Failures**
   - Check logs for error messages
   - Verify all paths in service files
   - Ensure virtual environments are activated
   - Check for conflicting audio device usage

3. **Wake Word Detection Issues**
   - Verify OpenWakeword service is running
   - Check network connectivity between services
   - Verify wake word configuration


# Add better LED control

Terminal
scp C:\Users\username\Documents\Github\pi-voice-assistant\config\2mic_service.py username@raspberrypi4:/home/username/wyoming-satellite/examples/2mic_service.py

Pi Shh
sudo systemctl restart 2mic_leds
