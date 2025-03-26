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
cd wyoming-satellite/
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

CInstall the wyoming satellite dependencies
```sh
python3 -m venv .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install \
  -f 'https://synesthesiam.github.io/prebuilt-apps/' \
  -r requirements.txt \
  -r requirements_audio_enhancement.txt \
  -r requirements_vad.txt
```
