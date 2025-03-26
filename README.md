# Wyoming-Satellite
Wyoming Satellite with External Bluetooth Speaker Connection

## Wyoming-Satellite Install Steps
Install the dependencies:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends  \
  git \
  python3-venv
```

Clone the `wyoming-satellite` repository:

```sh
git clone https://github.com/rhasspy/wyoming-satellite.git
```

If you have the ReSpeaker 2Mic or 4Mic HAT, recompile and install the drivers:

```sh
cd wyoming-satellite/
sudo bash etc/install-respeaker-drivers.sh
```

After install the drivers, you must reboot the satellite:

```sh
sudo reboot
```

Once the satellite has rebooted, reconnect over SSH and continue the installation:

```sh
cd wyoming-satellite/
python3 -m venv .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install \
  -f 'https://synesthesiam.github.io/prebuilt-apps/' \
  -e '.[all]'
```

If the installation was successful, you should be able to run:

```sh
script/run --help
```
