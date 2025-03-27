# Wyoming Satellite Installation with Home Assistant
This guide was created and verified on 03/26/2025 using a rasberry pi zero 2W, respeak 2mic board, and Home Assistant HASS io edition.

## Table of Contents
- [Update the Raspberry Pi](#update-the-raspberry-pi)
- [Install Git](#install-git)
- [Clone the Wyoming Satellite repository](#Clone-the-Wyoming-Satellite-repository)
- [Install the drivers for the Respeaker 2 Mic board](#Install-the-drivers-for-the-Respeaker-2-Mic-board)
- [Install the Wyoming Satellite dependencies](#Install-the-Wyoming-Satellite-dependencies)
- [Obtain your audio devices](#Obtain-your-audio-devices)
- [Add the Wyoming Satellite to Home Assistant](#Add-the-Wyoming-Satellite-to-Home-Assistant)
- [Create the wyoming satellite service](#Create-the-wyoming-satellite-service)
- [Install local wake word and its dependencies](#Install-local-wake-word-and-its-dependencies)
- [Create the wyoming local wake word service](#Create-the-wyoming-local-wake-word-service)
- [Enable the LED lights when you want to say something](#Enable-the-LED-lights-when-you-want-to-say-something)
- [Commands to reference](#Commands-to-reference)

## Update the rasberry pi
```sh
sudo apt-get update
```

## Install Git
```sh
sudo apt-get install --no-install-recommends  \
  git \
  python3-venv
```

## Clone the Wyoming Satellite repository
```sh
git clone https://github.com/rhasspy/wyoming-satellite.git
```

## Install the drivers for the Respeaker 2 Mic board

Change the directory to the wyoming satellite directory
```sh
cd wyoming-satellite/
```

Install the respeaker drivers
```sh
sudo bash etc/install-respeaker-drivers.sh
```

Reboot the OS
```sh
sudo reboot
```

## Install the Wyoming Satellite dependencies

Change the directory to the wyoming satellite directory
```sh
cd wyoming-satellite/
```

Install the wyoming satellite dependencies
```sh
python3 -m venv .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install \
  -f 'https://synesthesiam.github.io/prebuilt-apps/' \
  -e '.[all]'
```

Test to make sure the wyoming satellite dependencies are installed
```sh
script/run --help
```
An output of different help commands should be displayed. A failure would result in no help commands found.

## Obtain your audio devices

Find what your microphone device will be.  For Respeaker Mic 2 HAT, it will be the plughw:CARD=seeed2micvoicec,DEV=0
```sh
arecord -L
```

Record a test voice recording to a file on the rasberry pi.  This test will give you 5 seconds to say something before it automatically saves the file to the storage.
```sh
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav
```

Find what your speaker device will be.  For Respeaker Mic 2 HAT, it will be the plughw:CARD=seeed2micvoicec,DEV=0
```sh
arecord -L
```

Playback the voice recording to ensure the speaker is working correctly
```sh
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav
```

## Add the Wyoming Satellite to Home Assistant

Change the directory to the wyoming satellite directory
```sh
cd wyoming-satellite/
```

Add the wyoming satellite to home assistant
```sh
script/run \
  --debug \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw'
```

Open your home assistant instance and you should see the Wyoming Protocol "my satellite" waiting to be added as a device.  At the time of this writing, home assistant asks you to select the room you would like to add the device to and the voice assistant selection (cloud or local).  You may also see "Piper" and "Whisper" waiting to be aded to the Wyoming Protocol as well.  Proceed with adding Piper and Whisper to Home Assistant.

## Create the wyoming satellite service

Change the directory to the wyoming satellite directory
```sh
cd wyoming-satellite/
```

Create a new file for the service
```sh
sudo systemctl edit --force --full wyoming-satellite.service
```

Add the configuration and change "Pi" in the ExecStart line and the WorkingDirectory line to your rasberry pi username.
```sh
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run --name 'my satellite' --uri 'tcp://0.0.0.0:10700' --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' --mic-auto-gain 5 --mic-noise-suppression 2
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```
Save the file to memory

Enable the wyoming satellite service
```sh
sudo systemctl enable --now wyoming-satellite.service
```

Verify the wyoming satellite service is running without any errors
```sh
journalctl -u wyoming-satellite.service -f
```

If there were errors and you fixed them, restart the wyoming satellite service
```sh
sudo systemctl restart wyoming-satellite.service
```

## Install local wake word and its dependencies

Change the directory to the wyoming satellite directory
```sh
cd wyoming-satellite/
```

Update the rasberry pi 
```sh
sudo apt-get update
```

Install the local wake word dependencies
```sh
sudo apt-get install --no-install-recommends  \
  libopenblas-dev
```

Change to your home directory
```sh
cd ../
```

Clone the open wake word for Git
```sh
git clone https://github.com/rhasspy/wyoming-openwakeword.git
```

Change the directory to the wyoming open wake word directory
```sh
cd wyoming-openwakeword
```

Install the open wake word using the script file
```sh
script/setup
```

## Create the wyoming local wake word service

Create a new file for the service
```sh
sudo systemctl edit --force --full wyoming-openwakeword.service
```

Add the configuration and change "Pi" in the ExecStart line and the WorkingDirectory line to your rasberry pi username.
```sh
[Unit]
Description=Wyoming openWakeWord

[Service]
Type=simple
ExecStart=/home/pi/wyoming-openwakeword/script/run --uri 'tcp://127.0.0.1:10400'
WorkingDirectory=/home/pi/wyoming-openwakeword
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Update the wyoming satellite configuration file to to require the wyoming-openwakeword.serivce and also adding the wake word that will be used to activate the device when you want to say something.  Add the openwake word configuration within the "Unit" section. Add the wake-word-name configuration witin the "Service" section.  You can use "ok_nabu" or "hey_jarvis".
```sh
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target
Requires=wyoming-openwakeword.service

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run --name 'my satellite' --uri 'tcp://0.0.0.0:10700' --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' --mic-auto-gain 5 --mic-noise-suppression 2 --wake-uri 'tcp://127.0.0.1:10400' --wake-word-name 'ok_nabu'

WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Reload the services
```sh
sudo systemctl daemon-reload
```

Restart the wyoming satellite service
```sh
sudo systemctl restart wyoming-satellite.service
```

Verify the wyoming satellite and openwake word services are running within the "Active" line for each services.
```sh
sudo systemctl restart wyoming-satellite.service
```

## Enable the LED lights when you want to say something

Change to your home directory
```sh
cd ../
```

Change the directory to the wyoming satellite examples directory
```sh
cd wyoming-satellite/examples
```

Install the packages for the LEDs
```sh
python3 -m venv --system-site-packages .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install 'wyoming==1.5.2'
```

Install the necessary libraries
```sh
sudo apt-get install python3-spidev python3-gpiozero
```

Test to make sure the LED dependencies are installed
```sh
.venv/bin/python3 2mic_service.py --help
```
An output of different help commands should be displayed. A failure would result in no help commands found.

Create a new file for the service
```sh
sudo systemctl edit --force --full 2mic_leds.service
```

Add the configuration and change "Pi" in the ExecStart line and the WorkingDirectory line to your rasberry pi username.
```sh
[Unit]
Description=2Mic LEDs

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/examples/.venv/bin/python3 2mic_service.py --uri 'tcp://127.0.0.1:10500'
WorkingDirectory=/home/pi/wyoming-satellite/examples
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Update the wyoming satellite configuration file to to require the 2mic_LEDs.serivce and also adding the port to ensure the LEDs function.  Add the 2mic_LEDs configuration within the "Unit" section. Add the wake-word-name configuration witin the "Service" section.  You can use "ok_nabu" or "hey_jarvis".
```sh
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target
Requires=wyoming-openwakeword.service
Requires=2mic_leds.service

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run --name 'my satellite' --uri 'tcp://0.0.0.0:10700' --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' --mic-auto-gain 5 --mic-noise-suppression 2 --wake-uri 'tcp://127.0.0.1:10400' --wake-word-name 'ok_nabu' --event-uri 'tcp://127.0.0.1:10500'

WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Reload the services
```sh
sudo systemctl daemon-reload
```

Restart the wyoming satellite service
```sh
sudo systemctl restart wyoming-satellite.service
```

Verify the 2mic_LEDs service is running within the "Active" line.
```sh
sudo systemctl status wyoming-satellite.service 2mic_leds.service
```

Do a reboot of the rasberry pi as the LED lights may not work until this is done
```sh
sudo reboot
```

## Done
You can confirm that the wyoming satellite is installed and configured correct if you review the device in Home Assistant and say your wake word.  The voice assistant status should change from idle to active.

## Commands to reference

Check the status of the Wyoming Satellite, Openwake Word, and 2mic LED services
```sh
sudo systemctl status wyoming-satellite.service wyoming-openwakeword.service 2mic_leds.service
```

Check the active logs for the Wyoming Satellite, Openwake Word, and 2mic LED services
```sh
journalctl -u wyoming-satellite.service -f
```
```sh
journalctl -u wyoming-openwakeword.service -f
```
```sh
journalctl -u 2mic_leds.service -f
```

Review/Modify the configuration files for the Wyoming Satellite, Openwake Word, and 2mic LED
```sh
sudo systemctl edit --force --full wyoming-satellite.service
```
```sh
sudo systemctl edit --force --full wyoming-openwakeword.service
```
```sh
sudo systemctl edit --force --full 2mic_leds.service
```

Reboot the services after changes are made to any configuration file
```sh
sudo systemctl daemon-reload
sudo systemctl restart wyoming-satellite.service
```

Force start the Wyoming Satellite, Openwake Word, and 2mic LED services
```sh
sudo systemctl enable --now wyoming-satellite.service
sudo systemctl enable --now wyoming-openwakeword.service
sudo systemctl enable --now 2mic_leds.service
```
