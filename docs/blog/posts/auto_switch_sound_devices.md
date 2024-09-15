---
date: 2023-12-17
categories:
  - Ubuntu
  - Sound
  - PulseAudio
  - Article
tags:
    - Ubuntu
    - Sound
    - Article
---

# Auto-selects sound devices on login

## Motivation

The Ubuntu prioritizes the USB sound card over the internal sound card. This is not always what one wants.
Some examples:

1. USB camera with not-so-good microphone, when a better microphone is connected to the internal sound card.
2. USB microphone with option to monitor recorded sound, that is recognized as possible output, when speakers are connected to the internal sound card.

In both scenarios, the system will select the USB device instead of the internal sound card. This is not always what one wants.

<!-- more -->


## Solution

!!! note 

    This solution is tested on Ubuntu 22.04 with PulseAudio


Set the preferred sound card in the system settings. Then run `pactl info` to get the name of preferred sound device. 

```bash
$ pactl info
Server String: /run/user/1000/pulse/native
Library Protocol Version: 35
Server Protocol Version: 35
Is Local: yes
Client Index: 27
Tile Size: 65472
User Name: czaki
Host Name: grzesiek-komputer
Server Name: pulseaudio
Server Version: 15.99.1
Default Sample Specification: s16le 2ch 44100Hz
Default Channel Map: front-left,front-right
Default Sink: alsa_output.pci-0000_00_1f.3.analog-stereo
Default Source: alsa_input.pci-0000_00_1f.3.analog-stereo
Cookie: 057c:b811     
```

For single-line output one could use `pactl get-default-source` and `pactl get-default-sink` commands.

Then create a file `~/.local/bin/sound.sh` with the following content, add execution permissions (`chmod +x ./sound.sh`) and add it to the startup applications:

```bash
#!/bin/bash
pactl set-default-source 'alsa_input.pci-0000_00_1f.3.analog-stereo'
pactl set-default-sink 'alsa_output.pci-0000_00_1f.3.analog-stereo'
```

Replace target devices with the ones from `pactl info` output. 


## Additional scripts

Here is my script that allows one to switch between sound devices build based on above information.

If a Bluetooth device is connected then switch only the input device, otherwise switch both input and output devices between built-in and USB ones.
You may need to adjust the names of the devices to your needs.

```bash
#!/bin/bash

LANG=en_US  # to get consistent output from pactl

BUILDIN_OUTPUT=$(pactl list sinks | grep  -oP "Name: \K.*pci.*")
BUILDIN_INPUT=$(pactl list sources | grep  -oP "Name: \K.*input.*pci.*")
NCX_USB_OUTPUT=$(pactl list sinks | grep  -oP "Name: \K.*NCX.*")
NCX_USB_INPUT=$(pactl list sources | grep  -oP "Name: \K.*input.*NCX.*")
BLUEZ=$(pactl list sinks | grep  -oP "Name: \K.*bluez.*")


if [[ -n $BLUEZ ]]; then
    if [[ -n $NCX_USB_INPUT ]] && [[ $BUILDIN_INPUT == *$(pactl get-default-source)* ]]; then
        pactl set-default-source $NCX_USB_INPUT
    else
        pactl set-default-source $BUILDIN_INPUT
    fi
else
    if [[ -n $NCX_USB_INPUT ]] && [[ $BUILDIN_INPUT == *$(pactl get-default-source)* ]]; then
        pactl set-default-sink $NCX_USB_OUTPUT
        pactl set-default-source $NCX_USB_INPUT
    else
        pactl set-default-sink $BUILDIN_OUTPUT
        pactl set-default-source $BUILDIN_INPUT
    fi
fi
```


