# Wyoming Satellite Installation with Home Assistant
This guide was created and verified on 03/26/2025 using a rasberry pi zero 2W, respeak 2mic board, and Home Assistant HASS io edition.  At the time of the creation of this guide, all commands in this setup can be easily copied and pasted into your terminal without needing modifications. If you are using different mic or audio devices, pi devices, home assistant server, you may need to troubleshoot for any failures.  I do have this also setup with an OLLAMA docker contrainer running deepseek locally.  I do also have my satellites setup with an aggressive finished speaking detection and is incredibly fast at operating the commands.  I do not have any automations setup to use voice commands to operate my devices.

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
- [Additional Wyoming Satellite commands to use within your configuration file](#Additional-Wyoming-Satellite-commands-to-use-within-your-configuration-file)

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
I did get an error where it failed to fetch due to incorrect IP for some reason.  I rebooted the pi and ran the sudo-apt-get update again and it seems it was not pulling the latest version.  After running the apt-get update, i proceeded with the install git again and it pulled correctly.

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

You can skip over obtaining your audio devices if you are using the respeak device with your pi.  My commands include the plughw device for the configuration files later on.  If you do not get audio when finished with the entire setup, i would recommend coming back to this step and determining your audio devices.

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

Add the wyoming satellite to home assistant.  Make sure to change the --name value to what want the wyoming satellite to be called in Home Assistant.  You can always change this later too.
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

Add the configuration and change "Pi" in the ExecStart line and the WorkingDirectory line to your rasberry pi username.  This does add a sound everytime the wakeword is detected, it will also allow you to cancel the current comand and say a new command ASAP.
```sh
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run \
    --name 'my satellite' \
    --uri 'tcp://0.0.0.0:10700' \
    --mic-command 'arecord \
        -D plughw:CARD=seeed2micvoicec,DEV=0 \
        -r 16000 \
        -c 1 \
        -f S16_LE \
        -t raw' \
    --snd-command 'aplay \
        -D plughw:CARD=seeed2micvoicec,DEV=0 \
        -r 22050 \
        -c 1 \
        -f S16_LE \
        -t raw' \
        --snd-volume-multiplier .2 \
    --mic-auto-gain 5 \
    --mic-noise-suppression 2 \
    --wake-refractory-seconds 0 \
    --detection-command '/home/pi/wyoming-satellite/scripts/detection-script.sh'
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```
Ensure you also have the detection-script.sh file create, if not you can use the following steps to create it.

```sh
mkdir -p /home/pi/wyoming-satellite/scripts
nano /home/pi/wyoming-satellite/scripts/detection-script.sh
```

add 2 lines
```sh
#!/bin/bash
echo "Wake word detected. Interrupting current command."
```

Make the script executable
```sh
chmod +x /home/pi/wyoming-satellite/scripts/detection-script.sh
```

restart the wyoming satellite to apply changes
```sh
sudo systemctl restart wyoming-satellite.service
```

double check nothing is broken
```sh
journalctl -u wyoming-satellite.service --no-pager | tail -50
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

## Install local wake word and its dependencies. if you are going to use the wakeword service on the Rasberry pi.  Else, you can skip this step,

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

Install the open wake word using the script file.  It does take a few seconds before the code actually executes, so don't be alarmed and try to enter the code again.
```sh
script/setup
```

## Create the wyoming local wake word service. If you are going to use the wakeword service on the Rasberry pi.  Else, you can skip this step,

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
ExecStart=/home/pi/wyoming-openwakeword/script/run \
    --uri 'tcp://127.0.0.1:10400'
WorkingDirectory=/home/pi/wyoming-openwakeword
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Reopen the wyoming sattelite confiruation file to make updates
```sh
sudo systemctl edit --force --full wyoming-satellite.service
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
ExecStart=/home/pi/wyoming-satellite/script/run \
    --name 'my satellite' \
    --uri 'tcp://0.0.0.0:10700' \
    --mic-command 'arecord \
        -D plughw:CARD=seeed2micvoicec,DEV=0 \
        -r 16000 \
        -c 1 \
        -f S16_LE \
        -t raw' \
    --snd-command 'aplay \
        -D plughw:CARD=seeed2micvoicec,DEV=0 \
        -r 22050 \
        -c 1 \
        -f S16_LE \
        -t raw' \
        --snd-volume-multiplier .2 \
    --mic-auto-gain 5 \
    --mic-noise-suppression 2 \
    --wake-uri 'tcp://127.0.0.1:10400' \
    --wake-word-name 'ok_nabu' \
    --wake-refractory-seconds 0 \
    --detection-command '/home/pi/wyoming-satellite/scripts/detection-script.sh'
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
journalctl -u wyoming-openwakeword.service -f
```
```sh
journalctl -u wyoming-satellite.service -f
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
ExecStart=/home/pi/wyoming-satellite/examples/.venv/bin/python3 2mic_service.py \
    --uri 'tcp://127.0.0.1:10500'
WorkingDirectory=/home/pi/wyoming-satellite/examples
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Reopen the wyoming sattelite confiruation file to make updates
```sh
sudo systemctl edit --force --full wyoming-satellite.service
```

Update the wyoming satellite configuration file to to require the 2mic_LEDs.serivce and also adding the port to ensure the LEDs function.  Add the 2mic_LEDs configuration within the "Unit" section. Add the wake-word-name configuration witin the "Service" section.  You can use "ok_nabu" or "hey_jarvis".  Note, if you are not using the open wake word service on the wyoming satellite, omit the openwake word line under "Unit" and also the 2 lines "wake-uri" and "wake-word-name".
```sh
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target
Requires=wyoming-openwakeword.service
Requires=2mic_leds.service

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run \
    --name 'my satellite' \
    --uri 'tcp://0.0.0.0:10700' \
    --mic-command 'arecord \
        -D plughw:CARD=seeed2micvoicec,DEV=0 \
        -r 16000 \
        -c 1 \
        -f S16_LE \
        -t raw' \
    --snd-command 'aplay \
        -D plughw:CARD=seeed2micvoicec,DEV=0 \
        -r 22050 \
        -c 1 \
        -f S16_LE \
        -t raw' \
        --snd-volume-multiplier .2 \
    --mic-auto-gain 5 \
    --mic-noise-suppression 2 \
    --wake-uri 'tcp://127.0.0.1:10400' \
    --wake-word-name 'ok_nabu' \
    --event-uri 'tcp://127.0.0.1:10500' \
    --wake-refractory-seconds 0 \
    --detection-command '/home/pi/wyoming-satellite/scripts/detection-script.sh'
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Turn off the static green light when wakeword is not detected.  Make sure to change "pi" to your username.
```sh
nano /home/pi/wyoming-satellite/examples/2mic_service.py
```

Find the lines that say

```sh
 if StreamingStarted.is_type(event.type):
            self.color(YELLOW)
        elif Detection.is_type(event.type):
            self.color(_BLUE)
            await asyncio.sleep(1.0)  # show for 1 sec
        elif VoiceStarted.is_type(event.type):
            self.color(_YELLOW)
        elif Transcript.is_type(event.type):
            self.color(GREEN)
            await asyncio.sleep(1.0)  # show for 1 sec
        elif StreamingStopped.is_type(event.type):
            self.color(_BLACK)
        elif RunSatellite.is_type(event.type):
            self.color(_BLACK)
        elif SatelliteConnected.is_type(event.type):
```

You can change these colors to BLACK, WHITE, RED, YELLOW, BLUE, or GREEN.  Setting to streamingstarted to black keeps the lights off when no voice is detected, and changing the voicestarted to Blue will stay on while you speak so you know when you are done speaking, otherwise kept everything default expect the following changes... 
- I set "streamingStarted" to _BLACK
- I set "VoiceStarted" to _BLUE

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
## Additional Wyoming Satellite commands to use within your configuration file

show this help message and exit
```sh
  -h, --help
```

URI of Wyoming microphone service
```sh
--mic-uri MIC_UR
```

Program to run for microphone input
```sh
--mic-command MIC_COMMAND
```

Sample rate of mic-command (hertz, default: 16000)
```sh
--mic-command-rate MIC_COMMAND_RATE
```

Sample width of mic-command (bytes, default: 2)
```sh
--mic-command-width MIC_COMMAND_WIDTH
```

Sample channels of mic-command (default: 1)
```sh
 --mic-command-channels MIC_COMMAND_CHANNELS
```

Sample per chunk for mic-command (default: 1024)
```sh
  --mic-command-samples-per-chunk MIC_COMMAND_SAMPLES_PER_CHUNK
```
Mic volume multiplier
```sh
-mic-volume-multiplier MIC_VOLUME_MULTIPLIER
```

Mic noise suppression
```sh
--mic-noise-suppression {0,1,2,3,4}
```

Mic auto gain
```sh
  --mic-auto-gain {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31}
```

Seconds to mute microphone after awake wav is finished playing (default: 0.5)
```sh
--mic-seconds-to-mute-after-awake-wav MIC_SECONDS_TO_MUTE_AFTER_AWAKE_WAV
```

Don't mute the microphone while awake wav is being played
```sh
--mic-no-mute-during-awake-wav
```

Take microphone input from a specific channel (first channel is 0)
```sh
--mic-channel-index MIC_CHANNEL_INDEX
```

URI of Wyoming sound service
```sh
--snd-uri SND_URI
```

Program to run for sound output
```sh
--snd-command SND_COMMAND
```

Sample rate of snd-command (hertz, default: 22050)
```sh
--snd-command-rate SND_COMMAND_RATE
```

Sample width of snd-command (bytes, default: 2)
```sh
--snd-command-width SND_COMMAND_WIDTH
```

Sample channels of snd-command (default: 1)
```sh
--snd-command-channels SND_COMMAND_CHANNELS
```

Sound volume multiplier
```sh
--snd-volume-multiplier SND_VOLUME_MULTIPLIER
```  

URI of Wyoming wake word detection service
```sh
--wake-uri WAKE_URI
```  

Name of wake word to listen for and optional pipeline name to run (requires --wake-uri)
```sh
--wake-word-name name [pipeline ...]
```  

Program to run for wake word detection
```sh
--wake-command WAKE_COMMAND
```  

Sample rate of wake-command (hertz, default: 16000)
```sh
--wake-command-rate WAKE_COMMAND_RATE
```  

Sample width of wake-command (bytes, default: 2)
```sh
--wake-command-width WAKE_COMMAND_WIDTH
```  

Sample channels of wake-command (default: 1)
```sh
--wake-command-channels WAKE_COMMAND_CHANNELS
```  

Seconds after a wake word detection before another detection is handled (default: 5)
```sh
--wake-refractory-seconds WAKE_REFRACTORY_SECONDS
```  

Wait for speech before streaming audio
```sh
--vad
```  

Set the sensitivity level for Voice Activity Detection (VAD)
```sh
--vad-threshold VAD_THRESHOLD
```  

The number of consecutive frames of detected speech required to trigger the Voice Activity Detection (VAD)
```sh
--vad-trigger-level VAD_TRIGGER_LEVEL
```  

The length of audio (in seconds) stored in the buffer for Voice Activity Detection (VAD)
```sh
--vad-buffer-seconds VAD_BUFFER_SECONDS
```  

Seconds before going back to waiting for speech when wake word isn't detected
```sh
--vad-wake-word-timeout VAD_WAKE_WORD_TIMEOUT
```  

URI of Wyoming service to forward events to
```sh
--event-uri EVENT_URI
```  

Command run when the satellite starts
```sh
--startup-command STARTUP_COMMAND
```  

Command to run when wake word detection starts
```sh
--detect-command DETECT_COMMAND
```  

Command to run when wake word is detected
```sh
--detection-command DETECTION_COMMAND
```  

Command to run when speech to text transcript is returned
```sh
--transcript-command TRANSCRIPT_COMMAND
```  

Command to run when the user starts speaking
```sh
--stt-start-command STT_START_COMMAND
```  

Command to run when the user stops speaking
```sh
--stt-stop-command STT_STOP_COMMAND
```  

Command to run when text to speech text is returned
```sh
--synthesize-command SYNTHESIZE_COMMAND
```  

Command to run when text to speech response starts
```sh
--tts-start-command TTS_START_COMMAND
```  

Command to run when text to speech response stops
```sh
--tts-stop-command TTS_STOP_COMMAND
```  

Command to run when text-to-speech audio stopped playing
```sh
--tts-played-command TTS_PLAYED_COMMAND
```  

Command to run when audio streaming starts
```sh
--streaming-start-command STREAMING_START_COMMAND
```

Command to run when audio streaming stops
```sh
--streaming-stop-command STREAMING_STOP_COMMAND
```

Command to run when an error occurs
```sh
--error-command ERROR_COMMAND
```

Command to run when connected to the server
```sh
--connected-command CONNECTED_COMMAND
```

Command to run when disconnected from the server
```sh
--disconnected-command DISCONNECTED_COMMAND
```

Command to run when a timer starts
```sh
--timer-started-command TIMER_STARTED_COMMAND
```

Command to run when a timer is paused, resumed, or has time added or removed
```sh
--timer-updated-command TIMER_UPDATED_COMMAND
```

Command to run when a timer is cancelled
```sh
--timer-cancelled-command TIMER_CANCELLED_COMMAND, --timer-canceled-command TIMER_CANCELLED_COMMAND
```

Command to run when a timer finishes
```sh
--timer-finished-command TIMER_FINISHED_COMMAND
```

WAV file to play when wake word is detected
```sh
--awake-wav AWAKE_WAV
```

WAV file to play when voice command is done
```sh
-done-wav DONE_WAV
```

WAV file to play when a timer finishes
```sh
--timer-finished-wav TIMER_FINISHED_WAV
```

Number of times to play timer finished WAV and delay between repeats in seconds
```sh
--timer-finished-wav-repeat repeat delay
```

Specifies the Uniform Resource Identifier (URI) of a service that the system needs to communicate with (unix:// or tcp://)
```sh
--uri URI
```

Name of the satellite
```sh
-name NAME
```

Area name of the satellite
```sh
--area AREA
```

Disable discovery over zeroconf
```sh
--no-zeroconf
```

Name used for zeroconf discovery (default: MAC from uuid.getnode)
```sh
--zeroconf-name ZEROCONF_NAME
```

Host address for zeroconf discovery (default: detect)
```sh
--zeroconf-host ZEROCONF_HOST
```

Directory to store audio for debugging
```sh
--debug-recording-dir DEBUG_RECORDING_DIR
```

Log DEBUG messages
```sh
--debug
```

Format for log messages
```sh
--log-format LOG_FORMAT
```

Print version and exit
```sh
--version
```
