# Wyoming-Satellite
Wyoming Satellite Installation

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

Create a new file to add the configuration and change "Pi" in the ExecStart line and the WorkingDirectory line to your rasberry pi username.
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

