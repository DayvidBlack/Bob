# Bob — The Sentient Washing Machine (Raspberry Pi Voice Chatbot)

> **Availability & Kits (Important)**
>
> Bob is **free to make and print for personal use**. For convenience, a **limited number of pre‑printed kits** (including screens, switches, and microphones) are available here **[ominousindustries.com](https://ominousindustries.com/collections/robots/products/bob-the-sentient-washing-machine-parts-kit-no-pi-included)**.

> ![Bob kit photo](/imagebob.png)
---

## Contents

- [3D Printed Parts](#3d-printed-parts)
- [Step 1 — Audio, Bluetooth & Chatbot Base](#step-1--audio-bluetooth--chatbot-base)
  - [1. System packages](#1-system-packages)
  - [2. Enable services](#2-enable-services)
  - [3. Bluetooth speaker setup](#3-bluetooth-speaker-setup)
  - [4. USB microphone setup](#4-usb-microphone-setup)
  - [5. Project & Python deps](#5-project--python-deps)
  - [6. Install Ollama & model](#6-install-ollama--model)
  - [7. Create & run `chatbot.py`](#7-create--run-chatbotpy)
- [Step 2 — SPI Display (Waveshare) & “Bob” Chat](#step-2--spi-display-waveshare--bob-chat)
  - [1. Enable SPI & groups, reboot](#1-enable-spi--groups-reboot)
  - [2. Display packages (Pi 5 note)](#2-display-packages-pi-5-note)
  - [3. Waveshare driver](#3-waveshare-driver)
  - [4. Use project venv](#4-use-project-venv)
  - [5. Python deps for display](#5-python-deps-for-display)
  - [6. Test Waveshare demo](#6-test-waveshare-demo)
  - [7. Create `bobchat.py`](#7-create-bobchatpy)
  - [8. Link driver lib & sprites; run](#8-link-driver-lib--sprites-run)
- [Quick Reference](#quick-reference)
- [Notes](#notes)

---

## 3D Printed Parts

All print files are available here:  
**Printables:** <https://www.printables.com/model/1404129-bob-the-sentient-washing-machine>

A **limited number of pre‑printed kits** (including screens, switches, and microphones) are available here **[ominousindustries.com](https://ominousindustries.com/collections/robots/products/bob-the-sentient-washing-machine-parts-kit-no-pi-included)**.

---

## Step 1 — Audio, Bluetooth & Chatbot Base

### 1) System packages

~~~bash
# Update system packages
sudo apt update
sudo apt upgrade -y

# Core tools, audio stack, BT, and TTS
sudo apt install -y \
  git python3-venv python3-pip \
  python3-gpiozero python3-rpi.gpio \
  bluez \
  pipewire pipewire-pulse wireplumber \
  espeak-ng \
  libportaudio2 \
  alsa-utils \
  libspa-0.2-bluetooth

# Add user to audio group for device permissions
sudo usermod -aG audio $USER

# Reboot for group membership to take effect
sudo reboot
~~~

> After reboot, log back in and continue.

---

### 2) Enable services

~~~bash
# Enable Bluetooth system service
sudo systemctl enable --now bluetooth

# Start PipeWire audio stack for your user
systemctl --user enable --now pipewire wireplumber pipewire-pulse

# Keep user services running even when not logged in (headless)
sudo loginctl enable-linger $USER

# Verify PipeWire is up
systemctl --user status pipewire
~~~

---

### 3) Bluetooth speaker setup

~~~bash
# Enter the Bluetooth controller
bluetoothctl
~~~

Inside the `bluetoothctl` prompt, run:

~~~
power on
agent on
default-agent
scan on
# Wait until your speaker appears; note its MAC XX:XX:XX:XX:XX:XX

pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
exit
~~~

Set as default output and test:

~~~bash
# List PipeWire nodes; note your speaker's Sink ID
wpctl status

# Set default output (replace <sink-id> with the ID you noted)
wpctl set-default <sink-id>

# Test sound output
pw-play /usr/share/sounds/alsa/Front_Center.wav
~~~

---

### 4) USB microphone setup

~~~bash
# Plug in the USB mic, then list audio Sources
wpctl status

# Note the Source ID for your USB mic (e.g., 66) and set it as default:
wpctl set-default <source-id>

# Quick record test from the default source
pw-record --rate 44100 --channels 1 test.wav
# Speak ~3 seconds, then Ctrl+C

# Play back through default sink (BT speaker)
pw-play test.wav
~~~

---

### 5) Project & Python deps

~~~bash
# Create project
mkdir -p ~/voice-chatbot
cd ~/voice-chatbot

# Python virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Upgrade pip and install packages
pip install --upgrade pip
pip install faster-whisper ollama numpy kokoro

# Optional: thread tuning for Faster-Whisper (shell startup)
echo 'export OMP_NUM_THREADS=4' >> ~/.bashrc
source ~/.bashrc

# Ensure venv is active for the rest of the setup
source .venv/bin/activate
~~~

---

### 6) Install Ollama & model

~~~bash
# Install Ollama server
curl -fsSL https://ollama.com/install.sh | sh

# Enable and start the Ollama daemon
sudo systemctl enable --now ollama

# Download a small, fast model
ollama pull gemma3:270m

# Verify it responds
ollama run gemma3:270m "Say hello"
~~~

---

### 7) Create & run `chatbot.py`

~~~bash
# Create the chatbot script (paste in the chatbot.py script from the repo)
nano chatbot.py
~~~

Find your mic Source ID via `wpctl status`, then launch:

~~~bash
# Example: MIC_TARGET=66 (replace with your actual Source ID)
MIC_TARGET=66 python3 chatbot.py
~~~

---

## Step 2 — SPI Display (Waveshare) & “Bob” Chat

### 1) Enable SPI & groups, reboot

~~~bash
# Enable SPI in raspi-config:
sudo raspi-config   # Interface Options → SPI → Enable

# Add your user to GPIO/SPI groups
sudo usermod -a -G gpio,spi $USER

# Reboot to apply
sudo reboot
~~~

> After reboot, log back in and continue.

---

### 2) Display packages (Pi 5 note)

~~~bash
# On a Raspberry Pi 5, remove legacy RPi.GPIO first:
sudo apt purge -y python3-rpi.gpio
sudo apt autoremove -y

# Install required packages
sudo apt install -y python3-pip unzip wget git python3-lgpio
~~~

> If you are **not** on a Pi 5, you can skip the purge step.

---

### 3) Waveshare driver

~~~bash
cd ~
wget https://files.waveshare.com/upload/8/8d/LCD_Module_RPI_code.zip
unzip -o LCD_Module_RPI_code.zip
~~~

---

### 4) Use project venv

~~~bash
# Reuse the existing project venv (create only if it doesn't exist)
cd ~/voice-chatbot
[ -d .venv ] || python3 -m venv .venv
source .venv/bin/activate
~~~

---

### 5) Python deps for display

~~~bash
# Install display-related Python packages into the project venv
pip install pillow numpy spidev gpiozero rpi-lgpio lgpio

# (Optional) Use lgpio pin factory for gpiozero
export GPIOZERO_PIN_FACTORY=lgpio
~~~

---

### 6) Test Waveshare demo

~~~bash
# From inside the venv
cd ~/LCD_Module_RPI_code/RaspberryPi/python/example
python3 1inch28_LCD_test.py
~~~

---

### 7) Create `bobchat.py`

~~~bash
cd ~/voice-chatbot

# Create the Bob chat script (paste in the bobchat.py script from the repo)
nano bobchat.py

# Save and exit
~~~

---

### 8) Link driver lib & sprites; run

~~~bash
# Link Waveshare Python lib into the project for imports
ln -s "$HOME/LCD_Module_RPI_code/RaspberryPi/python/lib" "$HOME/voice-chatbot/lib"

# Sprites/images for Bob UI (optional but recommended)
git clone https://github.com/OminousIndustries/Bob_Images.git ~/voice-chatbot/sprites

# Run Bob chat (replace with your mic Source ID)
MIC_TARGET=<source-id> python3 bobchat.py
~~~

---

## Quick Reference

~~~bash
# Devices & routing
wpctl status
wpctl set-default <sink-or-source-id>

# Audio tests
pw-play /usr/share/sounds/alsa/Front_Center.wav
pw-record --rate 44100 --channels 1 test.wav

# Bluetooth pairing (inside bluetoothctl)
power on
agent on
default-agent
scan on
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
exit

# Core services
sudo systemctl enable --now bluetooth
systemctl --user enable --now pipewire wireplumber pipewire-pulse
sudo loginctl enable-linger $USER

# Ollama
sudo systemctl enable --now ollama
ollama pull gemma3:270m
ollama run gemma3:270m "Say hello"
~~~

---

## Notes

- Replace placeholders like `<sink-id>`, `<source-id>`, and `XX:XX:XX:XX:XX:XX` with values from `wpctl status` and `bluetoothctl`.
- Re-activate the venv in new shells: `source ~/voice-chatbot/.venv/bin/activate`.
- On Pi 5, prefer `lgpio`/`rpi-lgpio`; legacy `RPi.GPIO` is removed above to avoid conflicts.
